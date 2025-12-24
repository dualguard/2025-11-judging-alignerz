# [000587] Failure to Reset Claimed Seconds Between Distributions Causes Underflow Reverts Preventing Dividend Claims
  
  
## Summary

The failure to reset `dividendsOf[user].claimedSeconds` when starting a new dividend distribution will cause a permanent denial of service for users who partially claimed previous dividends, as the contract will consistently revert on underflow when calculating claimable seconds if the new vesting period is shorter than the previously claimed amount.
Note that there's no claiming window or deadline in A26Z contract, so it's normal for a user to not claim their remaining dividends for a long time.


## Root Cause

In [`A26ZDividendDistributor.sol:214-223`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L215-L218), the `_setDividends()` function sets new dividend amounts for users but never resets `dividendsOf[user].claimedSeconds` to zero. This causes `claimedSeconds` to persist across distributions, while `secondsPassed` in `claimDividends()` (lines 192-198) is calculated from zero up to the current `vestingPeriod`. When a user who partially claimed a previous distribution attempts to claim from a new distribution, the calculation `claimableSeconds = secondsPassed - claimedSeconds` (line 200) can underflow if `claimedSeconds` exceeds `secondsPassed`. This is especially problematic when the vesting period is decreased, as `secondsPassed` may never reach the stored `claimedSeconds` value, causing consistent reverts.

Additionally, in `A26ZDividendDistributor.sol:198`, the same underflow occurs when updating claimed seconds: `dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds)`.

## Internal Pre-conditions

1. At least one user needs to have partially claimed dividends from a previous distribution, resulting in `dividendsOf[user].claimedSeconds` being greater than zero
2. Admin needs to call `setUpTheDividends()` or `setDividends()` to start a new dividend distribution
3. The user's stored `claimedSeconds` value needs to be greater than or equal to the `secondsPassed` value that will be calculated when they attempt to claim (this is guaranteed if the user hasn't claimed for a while)
4. User attempts to call `claimDividends()` before `secondsPassed` exceeds their stored `claimedSeconds` value

## External Pre-conditions

None required.

## Attack Path

1. Admin calls `setVestingPeriod()` with a vesting period of 90 days (e.g., 7,776,000 seconds) and `setStartTime()` to initialize the first dividend distribution
2. Admin calls `setUpTheDividends()` to set dividend amounts for users
3. User calls `claimDividends()` after 50 days (4,320,000 seconds), resulting in `dividendsOf[user].claimedSeconds` being set to 4,320,000
4. User does not claim the remaining dividends from the first distribution (there is no deadline, so this is normal behavior)
5. Admin calls `setVestingPeriod()` again, optionally setting it to a lower value (e.g., 30 days or 2,592,000 seconds)
6. Admin calls `setStartTime()` to set a new start time for the new distribution
7. Admin calls `setUpTheDividends()` or `setDividends()` to start the new dividend distribution, which adds to `dividendsOf[user].amount` but does not reset `dividendsOf[user].claimedSeconds` (it remains 4,320,000)
8. User attempts to call `claimDividends()` to claim from the new distribution
9. The function calculates `secondsPassed = block.timestamp - startTime`, which ranges from 0 to the new `vestingPeriod` (at most 2,592,000 seconds if the period was decreased)
10. The function attempts to calculate `claimableSeconds = secondsPassed - claimedSeconds` (line 200), where `secondsPassed` ≤ 2,592,000 and `claimedSeconds` = 4,320,000
11. The subtraction underflows and the transaction reverts
12. The user cannot claim dividends until `secondsPassed` exceeds 4,320,000, which may never happen if the vesting period is shorter than the stored `claimedSeconds`

## Impact

Users who partially claimed previous dividends cannot claim dividends from new distributions. Their funds remain locked in the contract until either: (1) the vesting period is increased such that `secondsPassed` can exceed their stored `claimedSeconds`, or (2) the admin manually intervenes. This is a denial of service for affected users.

## Proof of Concept
N/A

## Mitigation

Reset `dividendsOf[user].claimedSeconds = 0` for all users when starting a new distribution. This can be done in the `_setDividends()` function by resetting the claimed seconds before or after setting new amounts. Alternatively, track claimed seconds per distribution period using a mapping with distribution identifiers.
  