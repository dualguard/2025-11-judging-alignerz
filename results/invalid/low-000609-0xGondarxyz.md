# [000609] User will lose their allocation in edge case regarding splitting
  
  ### Summary

In an edge case where a user aggressively splits NFT, at some point they will get zero allocation due to round down to zero.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132

### Details

If for any reason, a user splits multiple times via `splitTVS` function, after a certain point, the user will get 0 allocation and lose their allocation as a result. 

`splitTVS` calls `_computeSplitArrays` . In that function, there are these lines:

```jsx
        uint256[] memory baseAmounts = allocation.amounts;
   ...
   for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

**Remember that allocation shrinks on each split. And that there is no limit for splitting.**

after a certain point, baseAmount * percentage will be smaller than BASIS_POINT of 10_000, and the allocation returned from `_computeSplitArrays` will be zero.

Following that, back in `splitTVS` function, `_assignAllocation` will be called, and the new, zero allocation will be assigned.

As a result, user will lose their allocation.

### Impact

As likelihood of this occuring is low, and the effect is medium (user loses funds), I give it low severity. 

Can provide POC if the judge requires.
  