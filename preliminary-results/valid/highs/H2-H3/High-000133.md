# [000133] Multiple zero-length dynamic arrays cause deterministic reverts and mis-accounting
  
  ### Summary

Two helpers write into uninitialized dynamic memory arrays, causing immediate out-of-bounds panics whenever they are used with non-zero lengths. `_computeSplitArrays()` in `AlignerzVesting` writes into `alloc.*` arrays that are never allocated, and `FeesManager.calculateFeeAndNewAmountForOneTVS()` writes into `newAmounts` without allocating it. This is a **medium severity** issue because it breaks core protocol functionality

### Root Cause

- `_computeSplitArrays()` returns `Allocation memory alloc` and directly assigns `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, etc., without `new uint256[](nbOfFlows)` / `new bool[](nbOfFlows)`, so all writes target zero-length arrays and revert.  

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141

- `calculateFeeAndNewAmountForOneTVS()` declares `uint256[] memory newAmounts` but never initializes it to `new uint256[](length)`, so any `newAmounts[i]` write reverts.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- `splitTVS()` is completely non-functional for all real TVSs; users cannot split positions.  

- `mergeTVS()` also reverts at fee calculation, blocking merges.  

### Impact

- `splitTVS()` is completely non-functional for all real TVSs; users cannot split positions.  
- `mergeTVS()` also reverts at the fee calculation, blocking merges.  

### PoC

_No response_

### Mitigation

Allocate all dynamic arrays in memory helpers before writing:

```solidity
alloc.amounts = new uint256[](nbOfFlows);
alloc.vestingPeriods = new uint256[](nbOfFlows);
// ...
newAmounts = new uint256[](length);
```


  