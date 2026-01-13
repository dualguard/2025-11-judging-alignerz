# [000586] Decreasing Vesting Period After Partial Claims Causes Underflow Reverts Preventing Dividend Claims
  
  
## Summary

Decreasing the vesting period after users have partially claimed dividends will cause a permanent denial of service for affected users as the contract will consistently revert on underflow when calculating claimable seconds. Note that there's no claiming window or deadline in A26Z contract, so it's normal for a user to not claim their remaining dividends for a long time.

## Root Cause

In [`A26ZDividendDistributor.sol:200`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L198-L200), the calculation `claimableSeconds = secondsPassed - claimedSeconds` can underflow when the vesting period is decreased. The `claimedSeconds` value stored in `dividendsOf[user].claimedSeconds` (line 190) is based on a previous distribution's longer vesting period, while `secondsPassed` (lines 192-198) is calculated using the current (shorter) vesting period. When `claimedSeconds` exceeds `secondsPassed`, the subtraction reverts due to underflow.

Additionally, in `A26ZDividendDistributor.sol:198`, the same underflow issue occurs when updating the claimed seconds: `dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds)`.

```js
    function claimDividends() external {
        // ...
        if (block.timestamp >= vestingPeriod + startTime) {
            // ...
        } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }
        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        // ...
    }
```
## Internal Pre-conditions

1. Admin needs to call `setVestingPeriod()` to set `vestingPeriod` to a value lower than the vesting period used in a previous dividend distribution
2. At least one user needs to have partially claimed dividends from a previous distribution, resulting in `dividendsOf[user].claimedSeconds` being greater than zero and based on the previous (longer) vesting period
3. Admin needs to call `setUpTheDividends()` or `setDividends()` to start a new dividend distribution with the decreased vesting period
4. The user's stored `claimedSeconds` value needs to be greater than the `secondsPassed` value calculated using the new shorter vesting period

## External Pre-conditions

None required.

## Attack Path

1. Admin calls `setVestingPeriod()` with a vesting period of 90 days (e.g., 7,776,000 seconds)
2. Admin calls `setUpTheDividends()` to initialize the first dividend distribution
3. User calls `claimDividends()` after 50 days (4,320,000 seconds), resulting in `dividendsOf[user].claimedSeconds` being set to 4,320,000
4. Admin calls `setVestingPeriod()` again, setting it to 30 days (2,592,000 seconds), which is lower than the previous period
5. Admin calls `setStartTime()` to set a new `startTime` for the new distribution
6. Admin calls `setUpTheDividends()` to start the new dividend distribution
7. User attempts to call `claimDividends()` again
8. The function calculates `secondsPassed = block.timestamp - startTime`, which is at most 2,592,000 seconds (the new vesting period)
9. The function attempts to calculate `claimableSeconds = secondsPassed - claimedSeconds` (line 200), where `secondsPassed` ≤ 2,592,000 and `claimedSeconds` = 4,320,000
10. The subtraction underflows and the transaction reverts

## Impact

Affected users cannot claim dividends from new distributions. Their funds remain locked in the contract until the vesting period is increased to a value that makes `secondsPassed` greater than or equal to their stored `claimedSeconds. This is a permanent denial of service for affected users.

## Proof of Concept
N/A

## Mitigation

1. Reset `claimedSeconds` to zero when starting a new distribution, or
2. Validate that the new vesting period is not shorter than the previous one, or
3. Use a mapping to track claimed seconds per distribution period, or
4. Add a check in `claimDividends()` to handle the case where `claimedSeconds > secondsPassed` by resetting or adjusting the calculation accordingly
  