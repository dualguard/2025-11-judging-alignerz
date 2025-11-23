# [000814] Pool count not properly capped at 10
  
  ### Summary

`createPool()` intends to enforce a hard limit of 10 vesting pools per bidding project. However, the check uses:
```solidity
require(biddingProject.poolCount <= 10)
```
Since `poolCount` starts at 0, this actually allows values `0,1,2,...,10` , so 11 pools total.

### Root Cause

The check incorrectly uses a ≤ condition.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

Pools are not capped at 10 as intended.

### PoC

_No response_

### Mitigation

_No response_
  