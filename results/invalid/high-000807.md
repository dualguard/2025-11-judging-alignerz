# [000807] Users may be DoSed if dividend is not claimed in time
  
  ### Summary

`claimDividends()` calculates claimable dividends as:
```solidity
uint256 claimableSeconds = secondsPassed - claimedSeconds;
uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
```
`claimedSeconds` tracks the number of seconds already claimed by the user.
`vestingPeriod` is the current vesting duration for the allocation.

If a user has previously claimed **part of a dividend**, but **the vesting period has finished**. And the owner sets a new, **shorter vesting period** such that `claimedSeconds > vestingPeriod` then `claimDividends()` will revert due to an underflow here:
```solidity
uint256 claimableSeconds = secondsPassed - claimedSeconds;
```

### Root Cause

`claimedSeconds` will not be reset **if user does not claim after the vesting period**.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

User will be DoSed until `claimedSeconds > vestingPeriod` as `claimDividends()` will always revert.

### PoC

_No response_

### Mitigation

_No response_
  