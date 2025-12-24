# [000343] Writing to unallocated memory in AlignerzVesting.sol::`_computeSplitArrays` will corrupt memory or revert during TVS splits will miscompute allocations for all recipients
  
  ### Summary

When you declare an `Allocation memory alloc` and then try to assign values to its dynamic array members `(alloc.amounts[j], alloc.vestingPeriods[j], etc.)` inside the loop, the transaction will fail.

### Root Cause

In [`AlignerzVesting::_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) ,
the returned `Allocation memory alloc` has no memory arrays allocated. The code writes into `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, `alloc.vestingStartTimes[j]`, `alloc.claimedSeconds[j]`, and `alloc.claimedFlows[j]` without first doing `alloc.amounts = new uint256[](nbOfFlows)` (and same for the other arrays). Writing to `alloc.amounts[j]` when `alloc.amounts` has length 0 is invalid.

### Internal Pre-conditions

1. By calling the `splitTVS` function, it calls `_computeSplitArrays(...)` for `nbOfFlows > 0`.
2. The original `allocation.amounts` (and other arrays) have `nbOfFlows` entries.

### External Pre-conditions

THis is an internal logic bug

### Attack Path

1. The caller invokes `splitTVS` on a TVS that has `nbOfFlows > 0`.
2. `splitTVS` calls `_computeSplitArrays(...)`.
3. `_computeSplitArrays` attempts to assign value to `alloc.amounts[j]` but `alloc.amounts` was never allocated.
4. EVM either reverts with an invalid memory write or returns corrupted data

### Impact

This makes TVS splitting becomes unusable. Recipients receive incorrect allocations or the function reverts.
This also cause incorrect distributions of vested tokens, possible loss for TVS holders.

### PoC

_No response_

### Mitigation

```diff
        function _computeSplitArrays(...)
             internal
             view
             returns (Allocation memory alloc)
        {
   
                  // ...

+                alloc.amounts = new uint256[](nbOfFlows);
+                alloc.vestingPeriods = new uint256[](nbOfFlows);
+                alloc.vestingStartTimes = new uint256[](nbOfFlows);
+                alloc.claimedSeconds = new uint256[](nbOfFlows);
+                alloc.claimedFlows = new bool[](nbOfFlows);

                  for (uint256 j = 0; j < nbOfFlows;) {
                          //...
                          unchecked {
                               ++j;
                          }
                   }
    
}
```
  