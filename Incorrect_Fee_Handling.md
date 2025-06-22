# Incorrect Fee Handling on DoDo Router Swaps in `onCall` Function

## Summary

The `Gateway` can cause unintended fee bypass for users who perform swaps on the DoDo router, resulting in protocol revenue loss.

The missing fee deduction logic in the `onCall()` function causes incorrect fee accounting for the protocol. The `Gateway` contract can execute a DoDo Router-based swap where `_handleFeeTransfer` is skipped, leading to undercollection of platform fees.

## Root Cause

In `GatewayTransferNative.sol:357`, the function `onCall()` applies fee deduction via `_handleFeeTransfer()` only when `decoded.targetZRC20 == zrc20`.

In the swap path (`decoded.targetZRC20 != zrc20`), the full input amount is passed to `_doMixSwap()` with no fee deducted, resulting in incorrect net output.

The function is guarded by the `onlyGateway` modifier, so only the `Gateway` contract can invoke it. However, the bug still impacts all routed cross-chain swaps if using DoDo router.

## Internal Pre-conditions

- The `Gateway` contract routes a valid cross-chain call into `onCall()` of `GatewayTransferNative.sol`.
- The decoded message contains `decoded.targetZRC20 != zrc20` to follow the DoDo Router path.
- The `_handleFeeTransfer()` function is skipped because the code does not apply it in the DoDo router path.

## External Pre-conditions

- The `Gateway` contract is configured to support swaps via DoDo Router.
- The swap input and target tokens are different, causing routing to trigger `_doMixSwap()`.

## Attack Path

1. A cross-chain user requests a token swap from token A to token B (where `A ≠ B`).
2. `Gateway` decodes the request and routes it through `onCall()` in `GatewayTransferNative`.
3. Because the path involves a DoDo Router swap:

   - `_handleFeeTransfer` is not invoked.
   - Full amount is passed to `_doMixSwap()`, bypassing the protocol's intended 0.5% platform fee.

4. Swap result is forwarded to the user.
5. The event `EddyCrossChainSwap` logs a `platformFeesForTx` value that was never actually deducted.

## Impact

The protocol suffers consistent platform fee losses when swaps are routed through the DoDo route. The `Gateway`, though trusted, unintentionally facilitates systematic undercollection of fees, which may impact protocol revenue and tokenomics.

## Proof of Concept

```solidity
// Assume Gateway routes to onCall with swap from A to B (A != B)
// Fee is 0.5%, but not applied in DoDo swap branch

function testFeeBypassViaDoDoFromGateway() public {
    // simulate Gateway behavior
    MessageContext memory ctx = MessageContext(...);
    bytes memory message = encodeMessage(...); // Encodes A ≠ B, triggers _doMixSwap

    gatewayTransferNative.onCall(ctx, tokenA, 1000e18, message);

    // Expect: 0.5% fee deducted => 995e18 swap input
    // Actual: 1000e18 swap input (full amount), no fee collected
    assertEq(protocolFeeCollector.balance, 0);
}
```

## Mitigation

Refactor the fee logic to ensure that `_handleFeeTransfer()` is always applied before swap execution, regardless of token equality:

```solidity
uint256 platformFeesForTx = _handleFeeTransfer(zrc20, amount);
uint256 effectiveAmount = amount - platformFeesForTx;

if (decoded.targetZRC20 == zrc20) {
    TransferHelper.safeTransfer(decoded.targetZRC20, receiver, effectiveAmount);
} else {
    uint256 outputAmount = _doMixSwap(decoded.swapData, effectiveAmount, params);
    ...
}
```
