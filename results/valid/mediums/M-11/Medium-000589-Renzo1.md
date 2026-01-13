# [000589] Accumulation of Past Dividend Amounts with New Distribution Start Time Causes Fully Vested Dividends to be Treated as Still Vesting, Preventing Users from Claiming Past Distributions
  
  
## Summary

The accumulation of new dividend amounts onto existing dividend amounts without resetting the vesting tracking state will cause a denial of service for users as they will be unable to claim fully vested dividends from past distributions until the new distribution's vesting period elapses, because the claimable amount calculation treats all accumulated dividends as vesting from the new distribution's start time.

## Root Cause
```js
    function _setDividends() internal {
        // ...
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        // ...
    }

    function claimDividends() external {
        // ...
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
        // ...
    }
```
In [`A26ZDividendDistributor.sol:218`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218), new dividend amounts are accumulated onto existing amounts using `+=` without resetting the user's `claimedSeconds` or separating old and new distributions. In [`A26ZDividendDistributor.sol:187-201`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L188-L201), the `claimDividends()` function calculates claimable amounts using a single `startTime` and `vestingPeriod` for all accumulated dividends, treating fully vested dividends from past distributions as if they are still vesting from the current `startTime`.

When a new distribution occurs:
1. Line 218 adds new dividends to existing ones: `dividendsOf[owner].amount += ...`
2. The owner typically updates `startTime` to the new distribution time via `setStartTime()`
3. `claimedSeconds` is not reset and still reflects the old vesting period
4. Line 197 calculates `secondsPassed` from the new `startTime`, making it 0 or very small
5. Line 201 applies the vesting formula to the entire accumulated amount, treating old fully-vested dividends as if they started vesting from the new `startTime`

## Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` or `setDividends()` to add new dividend amounts to existing ones
2. Owner needs to call `setStartTime()` to update `startTime` to the new distribution timestamp (typically done before or after setting new dividends)
3. At least one user must have unclaimed dividends from a previous distribution where the vesting period has elapsed
4. The user's `dividendsOf[user].amount` must be greater than 0 from a previous distribution
5. The user's `dividendsOf[user].claimedSeconds` must be less than the previous distribution's vesting period (or the previous period must have fully elapsed)

## External Pre-conditions

None required.

## Attack Path

1. Owner calls `setUpTheDividends()` which internally calls `_setDividends()` at line 214
2. `_setDividends()` iterates through all NFT holders and accumulates new dividend amounts onto existing ones using `dividendsOf[owner].amount += ...` at line 218
3. Owner calls `setStartTime()` to update `startTime` to the current block timestamp (or the new distribution's start time)
4. User attempts to claim dividends by calling `claimDividends()`
5. The function retrieves `totalAmount` which includes both old fully-vested dividends and new dividends at line 189
6. The function retrieves `claimedSeconds` which still reflects the old vesting period at line 190
7. The function calculates `secondsPassed` from the new `startTime` at line 197, resulting in a value much smaller than the old `claimedSeconds`
8. The function calculates `claimableSeconds = secondsPassed - claimedSeconds` at line 200, which results in a negative or very small value (due to integer underflow or small positive value)
9. The function calculates `claimableAmount = totalAmount * claimableSeconds / vestingPeriod` at line 201, which treats all accumulated dividends (including fully-vested old ones) as if they're vesting from the new start time
10. User receives a minimal or zero claimable amount, unable to claim their fully-vested dividends from the previous distribution

## Impact

Users suffer a denial of service and cannot claim fully vested dividends from past distributions. They must wait for the new distribution's vesting period to elapse before they can claim both old and new dividends, even though the old dividends should be immediately claimable. This locks user funds for the duration of the new vesting period (typically 3 months), causing temporary loss of access to funds that should be available.

## Proof of Concept
N/A

## Mitigation

Separate tracking for each distribution or reset vesting state when new distributions are added. 
  