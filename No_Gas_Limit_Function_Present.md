# Summary

The absence of a setter function for the `gasLimit` variable in `GatewayCrossChain.sol` will cause a rigid and potentially unadaptive configuration for the protocol as the contract owner will be unable to update `gasLimit` after the initial `initialize` call.

# Finding Description

In `GatewayCrossChain.sol:35` there is no function defined to modify the `gasLimit` variable after it has been set in the `initialize` function.

## Internal Pre-conditions

- Owner calls `initialize()` to set `gasLimit` to a fixed value.
- No further call is possible to update `gasLimit` due to missing setter.

## External Pre-conditions

- Gas costs on the target chain increase beyond the initially configured `gasLimit`.
- Cross-chain transactions begin to fail due to insufficient gas forwarding.

# Attack Path

1. Owner deploys and configures `GatewayCrossChain.sol` via `initialize()` with an initial `gasLimit`.
2. Over time, the actual gas usage required for cross-chain operations increases.
3. Users initiate cross-chain transfers through the gateway.
4. These operations fail due to insufficient gas being allocated via the hardcoded `gasLimit`.
5. Users are either reverted or funds are stuck depending on the system architecture.

# Impact Explanation

**Severity: HIGH**

The protocol cannot execute cross-chain transactions once the gas cost exceeds the configured `gasLimit`. This leads to a partial or full halt of core functionality. Users are unable to complete operations, resulting in degraded user experience or loss of usability.

# Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

contract GatewayCrossChainTest is Test {
    GatewayCrossChain gateway;

    function setUp() public {
        gateway = new GatewayCrossChain();
        gateway.initialize(msg.sender, 200_000); // Sets gasLimit to 200,000
    }

    function testGasLimitUnchangeable() public {
        // No function exists to update gasLimit
        assertEq(gateway.gasLimit(), 200_000);
    }
}
```

# Recommendation

Add a `setGasLimit(uint256 newGasLimit)` function restricted to `onlyOwner`, similar to the existing implementation in `GatewaySend.sol` and `GatewayTransferNative.sol`, to allow post-deployment updates to the `gasLimit` value.

**Example Fix:**

```solidity
function setGasLimit(uint256 newGasLimit) external onlyOwner {
    gasLimit = newGasLimit;
}
```
