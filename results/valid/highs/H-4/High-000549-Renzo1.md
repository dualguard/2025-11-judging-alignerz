# [000549] Shared `claimedSeconds` Tracking Across Multiple Dividend Allocations in `_setDividends()` Breaks Vesting Mechanism for Topped-Up Dividends by Treating New Dividends as Partially Vested
  
  

## Summary

The use of `+=` to add new dividend amounts to existing allocations in `_setDividends()` combined with shared `claimedSeconds` tracking will cause an immediate unlock of topped-up dividends for users as the owner will add new dividends to existing allocations without resetting or separately tracking vesting progress, causing new dividends to be treated as if they have the same vesting progress as previously claimed dividends. An attacker can avoid claim their full dividend by a few seconds, so they can near instantly claim their next dividends.

```js
    function _setDividends() internal {
        // ...
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            // ...
    }
```
&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:218`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218), the `_setDividends()` function uses `+=` to add new dividend amounts to existing `dividendsOf[owner].amount` allocations. However, the `claimedSeconds` field in the `Dividend` struct is shared across all dividend amounts and does not differentiate between original and newly added dividends. When `claimDividends()` calculates the claimable amount in line 201 using `totalAmount * claimableSeconds / vestingPeriod`, it treats the entire `totalAmount` (which includes both old and new dividends) as having the same vesting progress as indicated by the shared `claimedSeconds` value. This causes newly added dividends to be immediately claimable in proportion to the already-claimed portion of the original dividends, breaking the vesting schedule for topped-up amounts.

&nbsp;

## Internal Pre-conditions

1. Owner needs to call `setDividends()` or `setUpTheDividends()` at least once to set initial dividend allocations
2. User needs to have claimed a portion of their dividends via `claimDividends()`, setting `claimedSeconds` to a value greater than zero but less than `vestingPeriod`
3. Owner needs to call `setDividends()` or `setUpTheDividends()` again to add additional dividends via `_setDividends()`
4. `stablecoinAmountToDistribute` needs to be greater than zero when the second `_setDividends()` call is made
5. `totalUnclaimedAmounts` needs to be greater than zero when the second `_setDividends()` call is made

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. Owner calls `setDividends()` or `setUpTheDividends()` to set initial dividend allocations, assigning 1000 USDC to a user with a 90-day vesting period
2. User waits 45 days and calls `claimDividends()`, receiving 500 USDC (50% of 1000 USDC) and setting `claimedSeconds` to 45 days
3. Owner calls `setDividends()` again to top up dividends, and `_setDividends()` adds 1000 USDC more to the user's allocation using `+=`, making `dividendsOf[user].amount` = 2000 USDC
4. The `claimedSeconds` value remains at 45 days, as it is not reset or tracked separately for the new dividends
5. User immediately calls `claimDividends()` again, and the calculation `claimableAmount = 2000 * (90 - 45) / 90 = 1000 USDC` treats the entire 2000 USDC as having 45 days already vested
6. User receives 1000 USDC immediately, which includes the full amount of the newly added 1000 USDC, breaking the vesting mechanism for the topped-up dividends

A strategic attacker can use this bug to unlock their subsequent dividend near instantly i.e. near zero vesting delay, at zero cost. All they have to do is leave a few seconds unclaimed in their first dividends claims. The next claim after the top up is then calculated as fully vested immediately it's distributed.

&nbsp;

## Impact

The protocol suffers a loss of vesting control over topped-up dividends. Users who have partially claimed their original dividends can immediately claim a proportional amount of newly added dividends without waiting for the full vesting period. For example, if a user has claimed 50% of their original dividends and the owner adds new dividends, the user can immediately claim 50% of the new dividends, effectively bypassing the intended vesting schedule for the topped-up amount. The protocol loses the ability to enforce proper vesting schedules when dividends are topped up.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation
Implement separate vesting tracking for each dividend allocation. 
  