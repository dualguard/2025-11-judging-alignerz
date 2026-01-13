# [000406] Off-By-One Error in Pool Limit Check Allows Creation of 11 Pools
  
  ### Summary

## Summary
The `createPool` function in `AlignerzVesting.sol` uses an incorrect comparison operator (`<=` instead of `<`) when checking the pool count limit, permitting creation of up to 11 pools per project instead of the intended maximum of 10. The error message "Cannot_Exceed_Ten_Pools_Per_Project" confirms the intent is a 10-pool limit.

Link to affected code: 
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

### Root Cause

The condition `require(biddingProject.poolCount <= 10)` uses `<=` when it should use `<`. Since `poolCount` starts at 0 and increments after each pool creation, the check allows pool IDs 0 through 10 (11 total pools) instead of the intended 0 through 9 (10 total pools).

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- `poolCount` starts at `0` when a project is launched.
- First pool created: `poolCount = 0`, check `0 <= 10` ✓, pool ID `0` assigned, `poolCount++` → `1`.
- ...
- Tenth pool created: `poolCount = 9`, check `9 <= 10` ✓, pool ID `9` assigned, `poolCount++` → `10`.
- **Eleventh pool** created: `poolCount = 10`, check `10 <= 10` ✓, pool ID `10` assigned, `poolCount++` → `11`.
- Twelfth pool attempt: `poolCount = 11`, check `11 <= 10` ✗, revert.

Result: Pool IDs `0` through `10` (11 total) can be created before hitting the limit.

### Impact

## Impact
- **Exceeds design specification**: Protocol documentation/error messages indicate a 10-pool limit; actual implementation permits 11, violating documented constraints.
- **Array/Merkle management complexity**: Downstream logic expecting exactly 10 pools may encounter issues if 11 pools exist (e.g., fixed-size off-chain Merkle tree construction, UI pagination, gas limit assumptions in loops over pools).


### PoC

_No response_

### Mitigation

Change the comparison operator from `<=` to `<` to enforce a strict upper limit of 10 pools (IDs `0-9`).

### Diff: Correct pool count limit check
```diff
@@ function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
-    require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+    require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

This ensures pool IDs `0` through `9` (10 total) can be created, aligning with the error message and documented design.
  