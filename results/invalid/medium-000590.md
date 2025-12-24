# [000590] Residual `claimedSeconds` from Previous Distributions Causes Permanent Token Loss in Future Dividend Claims
  
  
## Summary

The accumulation of `claimedSeconds` across multiple dividend distributions will cause a permanent loss of tokens for users who partially claim dividends, as the contract incorrectly assumes that previously claimed seconds apply to new distributions, resulting in undercounted claimable amounts that become permanent when the vesting period ends. 
Note that there's no claiming window or deadline in A26Z contract, so it's normal for a user to not claim their remaining dividends for a long time.

## Root Cause

In [`A26ZDividendDistributor.sol:218`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218), the `_setDividends()` function adds new dividend amounts to existing ones using `dividendsOf[owner].amount += ...` but does not reset `dividendsOf[owner].claimedSeconds`. This causes `claimedSeconds` from prior distributions to persist and be incorrectly applied to new distributions.
```js
    function _setDividends() internal {
        // ...
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        // ...
    }

    function claimDividends() external {
        // ...
        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
        // ...
    }
```
In [`A26ZDividendDistributor.sol:200`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L200), the calculation `uint256 claimableSeconds = secondsPassed - claimedSeconds;` subtracts the accumulated `claimedSeconds` from the current `secondsPassed`, incorrectly assuming that those seconds were already claimed for the new distribution. This undercounts the claimable seconds for users who partially claimed in previous distributions.

When the vesting period ends (lines 192-195), `dividendsOf[user]` is reset to zero, making any undercounted amounts permanently lost.

## Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` or `setDividends()` to add a new dividend distribution after a previous one has been partially claimed by users
2. At least one user needs to have partially claimed dividends from a previous distribution, leaving a non-zero `dividendsOf[user].claimedSeconds` value
3. The vesting period for the new distribution needs to be active (i.e., `block.timestamp >= startTime`)

## External Pre-conditions

None required. This is an internal accounting issue that occurs during normal contract operation.

## Attack Path

1. Owner calls `setUpTheDividends()` to set up the first dividend distribution, allocating dividends to users based on their unclaimed token amounts
2. User A calls `claimDividends()` and partially claims their dividends, resulting in `dividendsOf[userA].claimedSeconds` being set to, for example, 50 seconds
3. Owner calls `setUpTheDividends()` again to set up a second dividend distribution, which adds new amounts to `dividendsOf[userA].amount` using `+=` but leaves `dividendsOf[userA].claimedSeconds` at 50 seconds
4. User A calls `claimDividends()` again. The function calculates `claimableSeconds = secondsPassed - 50`, incorrectly assuming that 50 seconds of the new distribution have already been claimed
5. User A receives fewer tokens than they should, as the calculation undercounts their claimable seconds
6. When the vesting period ends, the `if` block at line 192 executes, resetting `dividendsOf[userA]` to zero, making the lost tokens permanently unrecoverable

## Impact

Users who partially claimed dividends from previous distributions suffer a permanent loss proportional to the residual `claimedSeconds` value. The loss increases with:
- The number of times new distributions are set up after partial claims
- The amount of `claimedSeconds` accumulated from previous partial claims
- The total dividend amount allocated in subsequent distributions

For example, if a user has `claimedSeconds = 100` from a previous distribution and a new distribution is set up with a vesting period of 90 days (7,776,000 seconds), they will lose approximately `(100 / 7,776,000) * totalDividendAmount` of their tokens, which becomes permanent when the vesting period ends.

## Proof of Concept
N/A

## Mitigation

Separate dividend distributions into sessions and track each session in isolation.
  