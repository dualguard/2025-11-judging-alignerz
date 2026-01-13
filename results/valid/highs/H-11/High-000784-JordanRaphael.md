# [000784] Precision Loss in Dividend Distribution Due to Rounding
  
  ### Summary

Accumulated rounding errors will cause users to suffer a loss of unclaimed dividends as the `A26ZDividendDistributor` will calculate claimable amounts using integer division that rounds down on each claim.

### Root Cause

In `A26ZDividendDistributor.claimDividends()` the calculation of `claimableAmount` uses integer division which rounds down the result. Specifically, at the line `uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;`, the multiplication and division operations perform rounding down, causing precision loss. When users claim dividends incrementally over time, each claim experiences rounding loss, and the accumulated losses prevent users from ever claiming the full entitled amount.
Respectively, `AlignerzVesting:claimTokens` function calculates the `claimableAmounts` the same way.

[claimDividends](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187-L204)

[AlignerzVesting claimableAmounts calculations](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L996)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User deposits funds and becomes eligible for dividends tracked in `dividendsOf[user].amount`
2. User calls `claimDividends()` at an arbitrary time during the vesting period
3. The function calculates `claimableSeconds` based on time elapsed and performs integer division: `totalAmount * claimableSeconds / vestingPeriod`
4. The result rounds down, losing fractional dividend amounts
5. User's `claimedSeconds` is incremented, but the lost fractional value is permanently unrecoverable
6. User repeats steps 2-5 throughout the vesting period, accumulating rounding losses with each claim


### Impact

Users suffer an approximate loss of dividend amounts due to cumulative rounding down in each claim transaction. The precise loss depends on the dividend amount, vesting period, and claim frequency, but can result in permanent loss of a portion of entitled dividends. 

### PoC

_No response_

### Mitigation

Track the cumulative claimed amount instead of calculating it incrementally. Maintain a state variable `claimedAmount` for each user and calculate their current entitled amount based on elapsed time, then transfer the difference between entitled and already-claimed amounts.
This ensures that at the last claim, the full entitled amount is claimed by the user.
  