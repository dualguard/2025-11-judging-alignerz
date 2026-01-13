# [000523] Off-by-one error in pool count validation allows creation of 11 pools instead of 10
  
  ### Summary

The AlignerzVesting::createPool function contains an off-by-one error in its pool count validation. The require statement uses <= instead of <, allowing the creation of 11 vesting pools per project when the design intent is to limit pools to 10.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L689
The validation check uses require(biddingProject.poolCount <= 10, ...) which permits poolCount values from 0 to 10 inclusive (11 total values). Since poolCount is used as the poolId for the new pool before being incremented, this allows the creation of pools with IDs 0 through 10, exceeding the intended maximum of 10 pools.

### Internal Pre-conditions

- The `BiddingProject` struct tracks `poolCount` as an incrementing counter
- Each call to `createPool` uses the current `poolCount` as the new pool's ID, then increments it
- The project must not be closed (`!biddingProject.closed`)


### External Pre-conditions

- The caller must be the contract owner (`onlyOwner`)
- Valid project ID must be provided
- The project must be in an open state

### Attack Path

No response

### Impact

The contract allows one more vesting pool to be created than intended. This could lead to:
- Inconsistency with documented pool limits
- Potential issues in downstream logic that assumes a maximum of 10 pools


### PoC

No response

### Mitigation

Change the validation operator from `<=` to `<` 
```diff
- require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+ require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
  