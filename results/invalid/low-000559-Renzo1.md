# [000559] State-Modifying Getter Function Breaks Off-Chain Querying and Front-End Integration Patterns
  
  
## Summary

The `getUnclaimedAmounts` function modifies state while being named as a getter, which prevents off-chain querying and breaks front-end integration patterns, as front-ends and off-chain tools cannot call it without sending a transaction and paying gas.

## Root Cause

In [`A26ZDividendDistributor.sol:140-161`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L160), the `getUnclaimedAmounts` function is declared as `public` (not `view`) and writes to storage at line 160:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... calculation logic ...
    unclaimedAmountsIn[nftId] = amount;  // Line 160: State modification
}
```

This violates the getter pattern: functions starting with "get" should be `view` or `pure` and not modify state. The state write prevents off-chain calls and requires gas, breaking standard front-end expectations.

## Internal Pre-conditions

1. A front-end or off-chain tool needs to query unclaimed amounts for an NFT.
2. The function is called directly or indirectly (e.g., via `getTotalUnclaimedAmounts` at line 127, which calls `getUnclaimedAmounts` at line 131).
3. The caller expects a read-only operation without state changes.

## External Pre-conditions

None required. This is a design issue that affects all callers.

## Attack Path

N/A

## Impact

Front-ends and off-chain tools cannot query unclaimed amounts without transactions and gas. This degrades UX, increases integration complexity, and raises gas costs when the function is called in loops (e.g., in `getTotalUnclaimedAmounts`). Users may see incorrect or stale data if the function isn't called, and the unexpected state changes can cause integration bugs.

## Proof of Concept
N/A

## Mitigation
Split into two functions:
- A `view` getter that only reads and returns the value.
- A separate function (e.g., `updateUnclaimedAmounts`) that writes to storage when needed.

  