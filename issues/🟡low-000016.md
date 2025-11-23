# [000016] Low: Off by one issue in `createPool`
  
  ### Summary

Per the error message in `AlignerzVesting.createPool(...)`, the protocol owner should not be able to create more than 10 pools. PoolId however starts counting from 0, so the cap of <=10 actually counts 11 pools.

### Root Cause

In `AlignerzVesting.createPool(...)`, `biddingProject.poolCount` is only incremented after creation of a pool, so `biddingProject.poolCount` can be 0.

```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  //<@ Counts 11
...
uint256 poolId = biddingProject.poolCount;
...
biddingProject.poolCount++; //<@ increments after setting poolId so first poolId is 0
```
Since: 
- poolCount starts at 0
- When poolCount == 10, the require still passes
You end up with 11 pools with IDs `0..10`, despite the error name saying “Cannot exceed ten pools per project”

### Internal Pre-conditions

1. `createPool` implements `require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());`

### External Pre-conditions

_No response_

### Attack Path

NIL

### Impact

A spec mismatch. 

### PoC

_No response_

### Mitigation

Cap to < 10
```solidity
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
  