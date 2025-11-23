# [000368] H10 Uninitialized memory in _computeSplitArrays causes DoS in splitTVS
  
  ## Summary

The `AlignerzVesting` contract contains a critical bug in `_computeSplitArrays` where it attempts to populate the arrays of a memory `Allocation` struct without allocating memory for them. This causes `splitTVS` to revert with an "Array Out of Bounds" panic, rendering the function unusable.

## Root Cause

In `AlignerzVesting.sol`:

```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    ) internal view returns (Allocation memory alloc) {
        // ...
        for (uint256 j; j < nbOfFlows; ) {
1. @>       alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

The return variable `alloc` is of type `Allocation memory`. When initialized, its dynamic array members (`amounts`, `vestingPeriods`, etc.) have a length of 0. The code does not perform `alloc.amounts = new uint256[](nbOfFlows)`, etc. Accessing `alloc.amounts[j]` at marker 1 causes a panic because the array has no allocated storage.

## Internal Pre-Conditions

1.  `splitTVS` is called (which calls `_computeSplitArrays`).
2.  `nbOfFlows > 0`.

## External Pre-Conditions

None.

## Attack Path

1.  User calls `splitTVS`.
2.  `_computeSplitArrays` is executed.
3.  The transaction reverts with `panic: array out-of-bounds access (0x32)`.

## Impact

**High**. The `splitTVS` function is broken and unusable.

## PoC

Trivial.

## Mitigation

Allocate memory for all dynamic arrays in the `alloc` struct before the loop.

```diff
    ) internal view returns (Allocation memory alloc) {
        // ...
+       alloc.amounts = new uint256[](nbOfFlows);
+       alloc.vestingPeriods = new uint256[](nbOfFlows);
+       alloc.vestingStartTimes = new uint256[](nbOfFlows);
+       alloc.claimedSeconds = new uint256[](nbOfFlows);
+       alloc.claimedFlows = new bool[](nbOfFlows);

        for (uint256 j; j < nbOfFlows; ) {
            // ...
        }
    }
```

  