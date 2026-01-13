# [001079] lack of input validation for some state variables results in division by zero
  
  ### Summary

When setting vesting period in `A26ZDividendDistributor::setVestingPeriod`, the function should make sure that `_vestingPeriod` is greater than zero because `vestingPeriod` is used in division. If its zero it will cause division by zero error.

### Root Cause
Insance-1
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L106

Here `vestingPeriod` is directly set based on the value passed by admin without any input validation. If its set to zero they it will lead to division by zero error in the following function

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187

Instance-2
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L460

### Internal Pre-conditions

1. Admin need to `A26ZDividendDistributor::setVestingPeriod` By passing zero as vesting period value

### External Pre-conditions

_No response_

### Attack Path

NA

### Impact

If the `vestingPeriod` is set to zero all the dividend claimings will be stopped due to division by zero error

### PoC

_No response_

### Mitigation

_No response_
  