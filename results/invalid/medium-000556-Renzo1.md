# [000556] Public `getUnclaimedAmounts()` function allows state manipulation leading to incorrect dividend distribution when owner calls `setDividends()` without recalculating amounts
  
  

## Summary

The public `getUnclaimedAmounts()` function mutates `unclaimedAmountsIn[]` state, and the separate `setDividends()` function uses this state without recalculating it. This allows an attacker to manipulate `unclaimedAmountsIn[]` values before the owner calls `setDividends()`, causing incorrect dividend distribution for TVS holders.
`allocationOf` values can be stale/incorrect as there are not updated in `AlignerzVesting::claimTokens` and `AlignerzVesting::mergeTVS`.

```js
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // ...
        unclaimedAmountsIn[nftId] = amount;
    }
```

## Root Cause

In [`A26ZDividendDistributor.sol:140-161`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L160), `getUnclaimedAmounts()` is public and mutates `unclaimedAmountsIn[nftId]` at line 160. In `A26ZDividendDistributor.sol:122-124`, `setDividends()` calls `_setDividends()` without recalculating `unclaimedAmountsIn[]` via `_setAmounts()`. The `_setDividends()` function at line 218 uses `unclaimedAmountsIn[i]` to calculate dividend allocations, relying on potentially stale or manipulated values.

Although, the owner can call `setUpTheDividends()` recalculate everything atomically, but the problem is that `setDividends()` exists as a **separate public function** that calls `_setDividends()` WITHOUT recalculating `unclaimedAmountsIn[]`. This creates an exploitation opportunity.

## Internal Pre-conditions

1. Owner must call `setDividends()` without first calling `setAmounts()` to recalculate `unclaimedAmountsIn[]`
2. `unclaimedAmountsIn[]` must contain stale or incorrect values from previous `getUnclaimedAmounts()` calls
3. At least one NFT must exist in the system for the manipulation to have effect

## External Pre-conditions

None required. The attack relies solely on internal contract state manipulation.

## Attack Path

1. Attacker calls `getUnclaimedAmounts(nftId)` for one or more specific NFT IDs, updating `unclaimedAmountsIn[nftId]` with values calculated at that point in time
2. Time passes or vesting state changes, making the stored `unclaimedAmountsIn[]` values stale
3. Owner calls `setDividends()` without first calling `setAmounts()` to recalculate `unclaimedAmountsIn[]`
4. `_setDividends()` uses the stale/incorrect `unclaimedAmountsIn[]` values to calculate dividend allocations (line 218)
5. Dividend distribution becomes incorrect, with some users receiving more or less than their fair share based on outdated calculations

## Impact

TVS holders suffer incorrect dividend distribution. The severity depends on how stale/incorrect the `unclaimedAmountsIn[]` values are and how many NFTs were affected. If an attacker manipulates values for specific NFTs before the owner calls `setDividends()`, those NFT holders may receive incorrect dividend amounts, while other holders' allocations are also affected due to the proportional distribution calculation using `totalUnclaimedAmounts`.

## Proof of Concept

N/A

## Mitigation
1. Remove the public `setDividends()` function entirely, forcing the owner to always use `setUpTheDividends()` which recalculates everything atomically
2. Consider making `getUnclaimedAmounts()` a view function that doesn't mutate state, and have a separate internal function to update `unclaimedAmountsIn[]` only when needed during the atomic `setUpTheDividends()` flow

  