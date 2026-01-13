# [000667] Users will receive excessive dividends across multiple distribution periods due to non-reset dividend amounts
  
  ### Summary

The failure to reset `dividendsOf[user].amount` between distribution periods causes users who do not fully claim their dividends to accumulate leftover amounts across multiple distribution periods. This results in them receiving more than their fair share, as the protocol incorrectly compounds unclaimed dividends with new distributions. Consequently, other users may receive less or even be unable to claim their entitled dividends.

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L196
In `claimDividends()`, whenever the vesting period is not yet completed, the user receives the correct proportional amount of dividends, but `dividendsOf[user].amount` is never reduced.
When the protocol start a new distribution period, `dividendsOf[user].amount`is not reset to `0` for the users who have not claim their dividend after `vestingPeriod + startTime`.
```solidity
  } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed -
                claimedSeconds);
        }
```        

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218
When `_setDividends()` is called, the dividends are acumulated between differents distribution Periods.
```solidity
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```
Since `dividendsOf[owner].amount` was never reset after the previous period ended, it accumulates leftovers across periods, making it incorrectly higher than it should be.
This causes dividend overpayment and creates inequality among users.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Lets say we have : `vestingPeriod = 30days`.
1. user1 calls `claimDividends()` after 29 days.
   -> He receives the dividends corresponding to the elapsed 29 days and `dividendsOf[user1].amount` doesnt change.
2. A new distribution period begins : the protocol calls:
    - `setStartTime()`
    - `setUpTheDividends()`
        - _`setDividends()` adds new dividends on top of the old, never-reset amount, causing `dividendsOf[user1]`.amount to become overestimated.
3. Later, user1 calls `claimDividends()` again.
    -> Because their accumulated `dividendsOf[user1].amount` is artificially inflated, user1 receives more than their fair share.


### Impact

- Users get more dividends than supposed to.
- Other users may lose dividends or even can't claim their dividends because the contract may run out of distributable funds.

### PoC

_No response_

### Mitigation

Reset `dividendsOf[].amount` to `0` for every user when you start a new period distribution.
  