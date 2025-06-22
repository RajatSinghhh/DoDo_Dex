# onCall() Not Marked Payable Causes msg.value to be Zero in Internal Swap Logic

## Summary

A function silently fails to execute expected ETH logic due to missing `payable` modifier.

The missing `payable` modifier in `onCall()` in `GatewayTransferNative.sol` will cause all ETH-based swap operations to fail silently or behave incorrectly, as `msg.value` will be `0` when passed to the internal `_doMixSwap` function. This can cause loss of expected output for users during swaps, especially when dealing with native ETH.

## Root Cause

In `GatewayTransferNative.sol`, the `onCall()` function is **not marked `payable`**. However, it invokes `_doMixSwap()`, which internally calls:

```solidity
IDODORouteProxy(DODORouteProxy).mixSwap{value: msg.value}(...);
```

Since `onCall()` is not `payable`, any ETH sent to it will cause the transaction to **revert**, or worse, `msg.value` will be **zero**, leading to failed swaps or incorrect behavior.

### Internal Pre-conditions

- A user sends ETH as part of the cross-chain message to trigger a swap using `onCall()`.
- The target swap operation requires ETH to be passed into `_doMixSwap()`.

### External Pre-conditions

- The external gateway sends a call to `onCall()` with native ETH.
- The DODO `_doMixSwap` expects a **non-zero `msg.value`**.

## Attack Path

1. Cross-chain message arrives at `onCall()` with a payload that includes a swap.
2. The contract attempts to execute `_doMixSwap()` with native ETH.
3. `msg.value` inside `_doMixSwap()` is `0`, due to missing `payable` in `onCall()`.
4. The call to `_doMixSwap{value: msg.value}` either:

   - Fails silently and returns zero output.
   - Reverts, breaking the entire chain of execution.

5. User receives zero output tokens or the protocol emits misleading swap events.

## Impact

- The user receives **no tokens** despite a successful event emission or deposit.
- The protocol **misrepresents** swap completion, causing trust and UX degradation.
- This could result in **griefing**, **loss of user funds**, or **broken flows** on the target chain.

**Affected Party**: Users interacting with ETH-based swap flow
**Loss Type**: Zero return / Swap failure due to missing ETH forwarding

## PoC

```solidity
// Assume a relayed cross-chain message triggers onCall()
// msg.value is expected to be non-zero for ETH-based swap, but isn't

gatewayTransferNative.onCall{value: 1 ether}(
    context,
    zrc20ETH,
    1 ether,
    message // swap message containing DODO mixSwap
);

// mixSwap will be called with msg.value = 0 and likely fail or revert
```

## Mitigation

Mark the `onCall()` function as `payable`:

```solidity
function onCall(
    MessageContext calldata context,
    address zrc20,
    uint256 amount,
    bytes calldata message
) external override onlyGateway payable {
    ...
}
```

This allows ETH to be passed correctly to internal functions such as `_doMixSwap()` which depend on `msg.value`.

**Always mark externally callable functions as `payable` if they pass `msg.value` to internal logic or external calls.**
