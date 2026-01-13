# [000094] Owner can create 11 pools instead of maximum 10 due to off-by-one error
  
  ### Summary

The incorrect boundary check in `createPool()` will allow the owner to create 11 pools per project instead of the intended maximum of 10, as the function checks `poolCount <= 10` instead of `poolCount < 10` before incrementing the counter.

### Root Cause

In `AlignerzVesting`, the `createPool()` uses `<=` instead of `<` in the pool count validation. Since poolCount starts at 0 and is used directly as the pool ID before being incremented, the check allows pool creation when `poolCount = 10`, resulting in pool ID 10 being created (the 11th pool, since IDs are 0-indexed).

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

### Internal Pre-conditions

1. Owner needs to call `launchBiddingProject()` to create a bidding project 
2. Owner needs to call `createPool()` 11 times for the same project

### External Pre-conditions

N/A

### Attack Path

1. Owner launches a bidding project via launchBiddingProject(), which initializes `poolCount = 0`
2. Owner calls `createPool()` 10 times successfully, creating pools with IDs 0-9
3. Owner calls `createPool()` an 11th time when poolCount = 10

- The check `poolCount <= 10` passes 
- Pool ID 10 is created 
- `poolCount` is incremented to 11 

4. The 12th call would fail since poolCount = 11 fails the <= 10 check

### Impact

- The protocol allows 11 pools per project instead of the documented maximum of 10 (violates the error message `Cannot_Exceed_Ten_Pools_Per_Project`).
- Could break off-chain systems or integrations that expet exaclty 10 polls maximum
- Owner when calling 'finalizeBids()`, he must provide merkle roots for all bids which could cause confusion if 11 roots are needed instead of 10

### PoC

_No response_

### Mitigation

Change the comparison operation for `<=` to `<`

```diff 
function createPool(...) external onlyOwner {
-      require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+     require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
.....
}
```
  