# [000269] `biddingProject.poolCount` can exceed `10` violating intended design
  
  ### Summary

In `AlignerzVesting.createPool()`, the check `biddingProject.poolCount <= 10` is used, which in practice allows the maximum pool count to reach `11`.

### Root Cause

In `AlignerzVesting.createPool()`, there is a require that is meant to enforce the “maximum ten pools per project” rule, and the error type name also reflects this intent:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project()); 
```
However, this condition uses `biddingProject.poolCount <= 10`. this allows the contract to reach a state where `biddingProject.poolCount` can become `11` while the check still passes during pool creation. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- The owner repeatedly calls `createPool()` for the same bidding project until the project’s poolCount exceeds the intended limit.

### Impact

A bidding project can create more than `10` pools, violating the supposed “ten pools per project” restriction.

### PoC

_No response_

### Mitigation

_No response_
  