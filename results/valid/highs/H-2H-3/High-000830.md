# [000830] Uninitialized memory arrays cause reverts in split and merge operations
  
  ### Summary

Accessing uninitialized dynamic memory arrays in `calculateFeeAndNewAmountForOneTVS()` and `_computeSplitArrays()` will cause all split and merge operations to revert as the functions attempt to write to arrays with zero length.

### Root Cause


In [FeesManager.sol:169-174](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the `newAmounts` array is never initialized before accessing its elements:

```solidity
function calculateFeeAndNewAmountForOneTVS(...) returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // newAmounts is uninitialized
    }
}
```

Similarly in [AlignerzVesting.sol:1113-1141](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) the `alloc` struct contains uninitialized dynamic arrays:

```solidity
function _computeSplitArrays(...) returns (Allocation memory alloc) {
    //...
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // alloc.amounts is uninitialized
        // ...
    }
}
```

When a dynamic array is declared in memory without initialization, its length is 0. Attempting to access any index causes an **out-of-bounds** revert.

### Internal Pre-conditions

Occur whenever user call `mergeTVS()` or `splitTVS()`

### External Pre-conditions

None

### Attack Path


This is not an attack but a critical bug that breaks core functionality:

1. User calls `splitTVS()` to split their vesting schedule
2. Function calls `calculateFeeAndNewAmountForOneTVS()` with the amounts array
3. Inside the loop, `newAmounts[0]` is accessed but `newAmounts.length` is 0
4. Transaction reverts with **out-of-bounds** error
5. User cannot split their TVS

Same issue occurs with `mergeTVS()` and any call to `_computeSplitArrays()`.

### Impact

All merge and split operations are completely broken and will always revert. Users cannot merge or split their vesting schedules, making these core protocol features unusable.

### PoC

_No response_

### Mitigation


Initialize the memory arrays with the correct length:

```diff
function calculateFeeAndNewAmountForOneTVS(...) returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);
    //...
}
```

```diff
function _computeSplitArrays(...) returns (Allocation memory alloc) {
+   alloc.amounts = new uint256[](nbOfFlows);
+   alloc.vestingPeriods = new uint256[](nbOfFlows);
+   alloc.vestingStartTimes = new uint256[](nbOfFlows);
+   alloc.claimedSeconds = new uint256[](nbOfFlows);
+   alloc.claimedFlows = new bool[](nbOfFlows);
    //...
}
```

  