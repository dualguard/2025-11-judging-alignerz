# [000875] `_computeSplitArrays()` will always revert due to unallocated dynamic arrays as struct elements.
  
  ### Summary

`_computeSplitArrays()` function returns `Allocation memory alloc` struct, which contains multiple dynamic arrays as elements:
```solidity
        uint256[] amounts; // Amount of tokens committed for this allocation for all flows
        uint256[] vestingPeriods; // Chosen vesting duration in seconds for all flows
        uint256[] vestingStartTimes; // start time of the vesting for all flows
        uint256[] claimedSeconds; // Number of seconds already claimed for all flows
        bool[] claimedFlows; // Whether flow is claimed
```
which all default to zero-length. Writing to any `alloc[array_element_name][j] will revert.

### Root Cause

`Allocation` struct array elements remain unallocated.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

`splitTVS()` is permanently broken.

### PoC

-

### Mitigation

Allocate dynamic arrays using `new type[](nbOfFlows)` for each struct element that is used for the return value. 
  