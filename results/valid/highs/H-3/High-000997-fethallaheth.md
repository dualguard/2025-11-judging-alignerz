# [000997] splitTVS always reverts due to uninitialized dynamic memory arrays
  
  ### Summary

The _computeSplitArrays function creates an Allocation memory alloc struct but never initializes its dynamic arrays (amounts, vestingPeriods, claimedFlows, etc.).
When the function tries to write to alloc.amounts[j], the contract reverts with an out-of-bounds memory panic.
As a result, splitTVS is completely unusable.

### Root Cause

Dynamic memory arrays must be created with new Type[](size), but the code does:
```solidity
Allocation memory alloc; // arrays uninitialized
alloc.amounts[j] = ...;  // always reverts
```
The code attempts to write to `alloc.amounts[j]` without initialization.

### Internal Pre-conditions

1. User calls `splitTVS`.

### External Pre-conditions

None.

### Attack Path

1. `splitTVS` calls `_computeSplitArrays`.
2. `_computeSplitArrays` tries to write to `alloc.amounts[0]`.
3. Revert (Panic: Array out of bounds / invalid memory access).

### Impact

The "Split TVS" feature is broken.

### PoC

_No response_

### Mitigation

Initialize the arrays in `_computeSplitArrays` before usage:
```solidity
alloc.amounts = new uint256[](nbOfFlows);
alloc.vestingPeriods = new uint256[](nbOfFlows);
// ... and so on
```
  