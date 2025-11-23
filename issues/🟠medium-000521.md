# [000521] Excessive Splitting in splitTVS Causes Permanent DoS on Claims
  
  ### Summary

Users can split NFTs repeatedly to create allocations with thousands of flows, making `claimTokens` exceed gas limits and rendering positions unclaimable.


### Root Cause

No limit on split count or flow array length; `claimTokens` loops over all flows.


### Internal Pre-conditions

- Split/merge fees low enough for repeated ops.

### External Pre-conditions

- User owns NFT with vested tokens.

### Attack Path

Split into max pieces, merge some, repeat to inflate flows.

### Impact

Permanent lock of user's (or transferred) tokens; griefing via transfer to victims.


### PoC

1. Start with 1-flow NFT.
2. Split into 100 (percentages 1/100 each).
3. Merge 99 into one, repeat – flows grow exponentially.
4. `claimTokens` hits gas limit.

### Mitigation

Add max flows limit:
```diff
+ require(allocation.vestingPeriods.length <= 50, "Too many flows");
```
  