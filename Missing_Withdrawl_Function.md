## Summary

The Owner will lose access to deposited tokens as the contract has no retrieval mechanism.

The missing `superWithdraw` function in `GatewaySend.sol` will cause a permanent loss of funds for users and maintainers as the contract lacks any mechanism to retrieve tokens deposited via `depositAndCall`.

---

## Root Cause

In `GatewaySend.sol:287`, the function `depositAndCall()` transfers ERC20 or ETH to the `GatewaySend` contract itself. However, unlike `GatewayTransferNative.sol` and `GatewayCrossChain.sol`, there is no `superWithdraw()` function or equivalent method to transfer those tokens out of the contract, resulting in stuck funds.

```solidity
function depositAndCall(
    address targetContract,
    uint256 amount,
    address asset,
    uint32 dstChainId,
    bytes calldata payload
) public payable {
    ...
    if (isETH) {
        ...
        _handleETHDeposit(...);
    } else {
        require(
            IERC20(asset).transferFrom(msg.sender, address(this), amount),
            "INSUFFICIENT AMOUNT: ERC20 TRANSFER FROM FAILED"
        );
        _handleERC20Deposit(...);
    }
    ...
}
```

In contrast, `superWithdraw()` is implemented in other gateway contracts:

```solidity
function superWithdraw(address token, uint256 amount) external onlyOwner {
    if (token == _ETH_ADDRESS_) {
        require(amount <= address(this).balance, "INVALID_AMOUNT");
        TransferHelper.safeTransferETH(EddyTreasurySafe, amount);
    } else {
        require(amount <= IZRC20(token).balanceOf(address(this)), "INVALID_AMOUNT");
        TransferHelper.safeTransfer(token, EddyTreasurySafe, amount);
    }
}
```

This absence leaves funds stuck in the contract forever.

---

## Internal Pre-conditions

- User calls `depositAndCall()` with a valid asset and amount.
- Tokens are successfully transferred to `GatewaySend.sol`.

## External Pre-conditions

- No upgrade or patch is deployed to add a withdraw or rescue function.
- The contract continues to receive deposits from users or integrations.

---

## Attack Path

- A legitimate user or integrator calls `depositAndCall()` with 1000 USDT.
- The contract transfers USDT from the user to `GatewaySend.sol`.
- Due to cross-chain or logical failure, no action occurs on the destination chain.
- The Owner attempts to recover the USDT.
- No function exists to retrieve funds from the contract.
- The 1000 USDT is permanently stuck in `GatewaySend.sol`.

---

## Impact

The protocol suffers a permanent loss of any ERC20 or ETH transferred to `GatewaySend.sol` through `depositAndCall` or similar entry points. The user or operator has no way to recover these assets due to missing withdrawal logic.

This impacts:

- Users depositing funds
- Devs debugging failed cross-chain operations
- Emergency rescue flows

---

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract GatewaySendPoC {
    address gatewaySend = 0x...; // deployed GatewaySend address
    address token = 0x...;       // address of test ERC20
    uint256 amount = 1_000e6;

    function testDeposit() public {
        IERC20(token).approve(gatewaySend, amount);
        IGatewaySend(gatewaySend).depositAndCall(
            address(0),
            amount,
            token,
            1001,
            "0x"
        );
        // Tokens are now stuck in GatewaySend.sol
    }

    function testWithdrawFails() public {
        // No way to call `superWithdraw()` or retrieve the tokens
    }
}
```

---

## Mitigation

Add a `superWithdraw()` function in `GatewaySend.sol`, following the exact access control and logic from `GatewayTransferNative.sol`.

### Example:

```solidity
function superWithdraw(address token, uint256 amount) external onlyOwner {
    if (token == _ETH_ADDRESS_) {
        require(amount <= address(this).balance, "INVALID_AMOUNT");
        TransferHelper.safeTransferETH(EddyTreasurySafe, amount);
    } else {
        require(amount <= IZRC20(token).balanceOf(address(this)), "INVALID_AMOUNT");
        TransferHelper.safeTransfer(token, EddyTreasurySafe, amount);
    }
}
```

This function should be `onlyOwner` or protected with equivalent access control.
