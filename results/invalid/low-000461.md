# [000461] Mistake in the condition to create maximum 10 pools
  
  ### Summary

`createPool` has a check that reverts if we try to create more than 10 pools. However, the condition make it revert only if we try to make more than 11 pools.

### Root Cause

`biddingProject.poolCount <= 10` should be `biddingProject.poolCount < 10`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Create more than 10 pools

### Impact

More pools are made than what expected

### PoC

_No response_

### Mitigation

```diff
-   biddingProject.poolCount <= 10
+   biddingProject.poolCount <  10
```
  