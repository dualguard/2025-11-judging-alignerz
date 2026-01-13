# [001003] check Error allows creating more pools than intended
  
  ### Summary

The check `biddingProject.poolCount <= 10` allows the creation of an 11th pool (index 10), violating the intended limit of 10 pools.


### Root Cause

In `AlignerzVesting.sol:830`:
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
The `poolCount` starts at 0.
- When `poolCount` is 10, the check `10 <= 10` passes.
- A new pool is created at index 10.
- `poolCount` increments to 11.
- Total pools created: 0 to 10 = 11 pools.


### Internal Pre-conditions

_No response_

### External Pre-conditions

None

### Attack Path

1. Owner calls `createPool` repeatedly.
2. The 11th call succeeds, creating an 11th pool.

### Impact

The protocol supports slightly more pools than intended.

### PoC

_No response_

### Mitigation

Change the check to `require(biddingProject.poolCount < 10, ...);`.
  