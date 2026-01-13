# [000787] ComputeSplitArrays Always Reverts
  
  ### Summary

An implementation error will cause a complete failure of vesting split allocation for the vesting contract as the vesting logic will attempt to write to unallocated memory.

### Root Cause

In `AlignerzVesting.sol:_computeSplitArrays`, the function attempts to assign values to dynamic array members of the return parameter `alloc` without first allocating memory for those arrays. The `Allocation memory alloc` struct is declared but its dynamic array fields (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) are never initialized via memory allocation, causing the assignments in the loop to fail with an out-of-bounds memory access error.

[assignment to unallocated memory](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132-L1136)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A user or contract calls any function that invokes `_computeSplitArrays` with valid parameters
2. The function executes and reaches the for loop
3. The loop attempts to assign values to `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, and other array fields
4. Since these arrays were never allocated in memory, the transaction reverts


### Impact

The vesting contract cannot execute any vesting split allocation operations. Users cannot successfully claim or manage split vesting allocations, resulting in complete unavailability of core vesting functionality that depends on this function.


### PoC

_No response_

### Mitigation

Allocate memory for the dynamic arrays in the `alloc` struct before attempting to assign values. 
  