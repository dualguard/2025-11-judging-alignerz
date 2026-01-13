# [000873] 11 pools can be created per bidding project, instead of the max 10 
  
  ### Summary

According to the following custom error, each bidding project should have at most 10 pools assigned with `createPool()` function:
```solidity
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  
```
However, because the check is **inclusive**, it does not revert when `poolCount` is already at 10 - allowing the 11-th pool to be created.

### Root Cause

`poolCount <= 10` instead of `poolCount < 10`

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

More pools can be created than intended.

### PoC

-

### Mitigation

Replace `<=` with `<`. 
  