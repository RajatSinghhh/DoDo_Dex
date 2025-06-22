# ETH Transfer via \_handleETHDeposit() Fails Due to Payability Mismatch

## Summary

An incorrect low-level call in `GatewaySend.sol:277` will cause a transaction failure for users as the contract sends native ETH to a function (`depositAndCall`) that is not marked as `payable`, which will revert the transaction when invoked.

## Root Cause

In `GatewaySend.sol`, the `_handleETHDeposit()` function performs a call to `gateway.depositAndCall{value: amount}(...)`. However, the target function `depositAndCall()` defined in the gateway contract is **not marked as payable**, and thus **cannot receive ETH** via `{value: amount}`.

This results in the **entire transaction reverting** when ETH is sent, even if all other parameters are correct.

## Internal Pre-conditions

- User triggers a function that routes to `_handleETHDeposit()` with `amount > 0`
- Internal routing leads to execution of `_handleETHDeposit()`

## External Pre-conditions

- The `gateway.depositAndCall()` function remains non-payable
- Native ETH is used as the transfer medium instead of ZRC20 tokens

## Attack Path

1. User or dApp triggers a deposit flow via `GatewaySend` for ETH
2. `_handleETHDeposit()` attempts to send native ETH to a non-payable function
3. Transaction reverts with an exception (e.g., non-payable function error)

This leads to **DoS (Denial of Service)** for ETH-based flows via this route

## Impact

- The user **cannot complete ETH deposit transactions** via this route.
- The protocol becomes **unusable for ETH deposits** unless the underlying call is corrected.
- No funds are lost, but all related operations will revert, leading to **service disruption**.

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface IGateway {
    function depositAndCall(
        address target,
        bytes calldata message,
        RevertOptions calldata revertOptions
    ) external;
}

struct RevertOptions {
    address revertAddress;
    bool callOnRevert;
    address abortAddress;
    bytes revertMessage;
    uint256 onRevertGasLimit;
}

contract Exploit {
    IGateway public gateway;

    constructor(address _gateway) {
        gateway = IGateway(_gateway);
    }

    function triggerRevert(address targetContract) external payable {
        RevertOptions memory revertOptions = RevertOptions({
            revertAddress: address(this),
            callOnRevert: true,
            abortAddress: address(0),
            revertMessage: "0x",
            onRevertGasLimit: 50000
        });

        // This call will revert because depositAndCall is not payable
        gateway.depositAndCall{value: msg.value}(
            targetContract,
            "0xdeadbeef",
            revertOptions
        );
    }
}
```

## Mitigation

- **Do not send ETH** using `{value: amount}` to `_handleETHDeposit()` if `gateway.depositAndCall()` is not marked as `payable`
- Instead, pass `amount` as a function parameter just like other values
- Alternatively, **mark `depositAndCall` as payable** in the receiving contract to allow safe ETH transfers
