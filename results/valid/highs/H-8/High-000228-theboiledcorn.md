# [000228] Infinite loop in `getUnclaimedAmounts` causes permanent Denial of Service
  
  ### Summary

The missing loop increment in `getUnclaimedAmounts` will cause a permanent Denial of Service for the protocol and TVS holders as the Admin will trigger an infinite loop when attempting to calculate unclaimed amounts if **any** single vesting flow satisfies the skip condition, resulting in an Out of Gas revert.

### Root Cause

In [`A26ZDividendDistributor.sol:147`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147), the `for` loop is defined without an update expression in the header:

```solidity
for (uint i; i < len;) {
```

Inside this loop, the `continue` keyword is used in two conditional checks. When executed, `continue` jumps to the update expression of the loop. Since the update expression is empty, the manual increment `unchecked { ++i; }` at the end of the loop body is skipped.

```solidity
// A26ZDividendDistributor.sol

for (uint i; i < len;) {
    if (claimedFlows[i]) continue; // @audit-issue Jump to empty header, i is not incremented
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue; // @audit-issue Jump to empty header, i is not incremented
    }
    // ... logic ...
    unchecked {
        ++i; // This is skipped when continue is hit
    }
}
```

This causes the loop to restart with the same index `i`, triggering the same condition repeatedly until the transaction runs out of gas. This applies regardless of whether the issue occurs at the first index (`0`) or the last index (`len - 1`).

### Internal Pre-conditions

1.  A user must hold an NFT that has an active vesting allocation in `AlignerzVesting.sol`.
2.  The allocation array must contain **at least one** index `i` where:
      * `claimedFlows[i]` is `true`.
      * **OR** `claimedSeconds[i]` is `0`.

### External Pre-conditions

None.

### Attack Path

1.  **Admin** calls `setUpTheDividends()` or `setAmounts()`.
2.  The contract calls `getTotalUnclaimedAmounts()`, which calls `getUnclaimedAmounts(nftId)`.
3.  `getUnclaimedAmounts` retrieves the vesting arrays (e.g., `claimedSeconds`, `amounts`) from `AlignerzVesting`.
4.  The function begins iterating through the arrays using index `i`.
5.  If `i=0` is valid (not claimed, `claimedSeconds > 0`), the code executes normally, increments `i` to `1`, and continues.
6.  The loop reaches an index `k` (where `k < len`) that satisfies the skip condition (e.g., a new vesting flow where `claimedSeconds[k] == 0`).
7.  The `continue` statement executes.
8.  The execution jumps to the header without incrementing `i`. `i` remains equal to `k`.
9.  The loop repeats the check for index `k`, sees the same condition, and loops indefinitely.
10. The transaction reverts due to **Out of Gas**.

### Impact

The protocol cannot calculate or distribute dividends. The functions `setUpTheDividends`, `setAmounts`, and `getTotalUnclaimedAmounts` are permanently broken. TVS holders cannot receive any rewards if they have even a single "unstarted" or "fully claimed" vesting flow.

### PoC

1.  **Setup**: An `AlignerzVesting` contract exists with a user holding `NFT ID 1`. The NFT has an allocation with **3 vesting flows**:
      * Flow 0: Started and partially claimed (`claimedSeconds > 0`).
      * Flow 1: Started and partially claimed (`claimedSeconds > 0`).
      * Flow 2: Newly added flow, `claimedSeconds` is `0`.
2.  **Execution**: The Admin calls `A26ZDividendDistributor.getTotalUnclaimedAmounts()`.
3.  **Iteration 0**: The loop checks Flow 0. Logic processes. `i` increments to 1.
4.  **Iteration 1**: The loop checks Flow 1. Logic processes. `i` increments to 2.
5.  **Iteration 2**: The loop checks Flow 2. It detects `claimedSeconds[2] == 0`.
6.  **The Trigger**: The `continue` statement is executed. The manual increment `unchecked { ++i; }` is skipped.
7.  **The Loop**: The code jumps to the header. `i` is still 2.
8.  **Infinite Cycle**: The loop re-evaluates Flow 2, finds `claimedSeconds` is still 0, and repeats step 6-7 until gas is exhausted.

### Mitigation

Move the increment logic `++i` into the `for` loop header. This ensures the loop counter advances even when `continue` is executed.

```solidity
// Recommended Fix
for (uint i; i < len; ++i) { // Increment in header
    if (claimedFlows[i]) continue;
    
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue;
    }
    
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    
    // Remove: unchecked { ++i; }
}
```
  