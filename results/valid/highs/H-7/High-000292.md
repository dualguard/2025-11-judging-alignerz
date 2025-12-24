# [000292] mergeTVS() & splitTVS() will never execute because of an infinite loop in the function calculating fees
  
  ### Summary

calculateFeeAndNewAmountForOneTVS() is a function called only by mergeTVS() and splitTVS() and it contains an infinite for loop that iterates over and over with the same input without increment.

### Root Cause

The for loop in function FeesManager.sol:calculateFeeAndNewAmountForOneTVS() never increases `i`
https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170-L173

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user tries to execute mergeTVS() or splitTVS()
2. infinite loop

### Impact

Completely blocking splitting and merging of the TVS, with a 100% likelihood

### PoC

_No response_

### Mitigation

Do i++
  