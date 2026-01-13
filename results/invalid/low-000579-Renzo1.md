# [000579] Off-by-One Error in Pool Count Validation Allows Creation of Eleven Pools Instead of Ten in `AlignerzVesting::createPool`
  
  
## Summary

An off-by-one error in `AlignerzVesting::createPool` allows 11 pools (IDs 0-10) instead of 10. The check uses `<= 10` instead of `< 10`, so when `poolCount` is 10, the check passes and an 11th pool is created, violating the intended limit.


## Root Cause

In [`AlignerzVesting.sol:690`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689), the check uses `<= 10` instead of `< 10`. Since `poolCount` starts at 0 and is incremented after pool creation, when `poolCount` is 10, the check passes and pool ID 10 is created, resulting in 11 pools (IDs 0-10).
```js
    function createPool(...)
        external
        onlyOwner
    {
        // ...
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  

        // ...
    }
```

## Internal Pre-conditions

1. Owner must call `launchBiddingProject` to create a bidding project with `poolCount` initialized to 0
2. Owner must call `createPool` 10 times to set `poolCount` to 10
3. Owner must call `createPool` an 11th time while `poolCount` equals 10

## External Pre-conditions

None

## Attack Path

1. Owner calls `launchBiddingProject` to create a bidding project (line 669: `poolCount = 0`)
2. Owner calls `createPool` 10 times, incrementing `poolCount` from 0 to 10 (lines 694-700)
3. Owner calls `createPool` an 11th time; at line 690, `poolCount` is 10, so `poolCount <= 10` passes
4. The function creates pool ID 10 (line 694), resulting in 11 pools (IDs 0-10)

## Impact

The protocol allows 11 pools per project instead of 10, violating the intended limit. This can affect allocation logic, gas costs, and downstream systems that assume a maximum of 10 pools.

## Proof of Concept
N/A

## Mitigation

Change line 690 to:
```js
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
  