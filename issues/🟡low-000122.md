# [000122] Self-merge of TVS can permanently lock vesting flows
  
  ### Summary

`mergeTVS()` does not prevent `mergedNftId` from also appearing in the `nftIds` array. If a user passes the same NFT as both the merge target and a source, the function will effectively merge a TVS into itself, corrupt its allocation, and then burn the NFT. This is a low-severity issue because it is self-inflicted and does not create extra claimable amounts, but it can permanently lock a user’s vesting flows.

### Root Cause

`mergeTVS()` blindly iterates over nftIds and calls `_merge(mergedTVS, projectIds[i], nftIds[i], token)` without checking `nftIds[i] != mergedNftId`. Inside `_merge()`, when `nftId == mergedNftId`, both `mergedTVS` and `TVSToMerge` reference the same storage allocation, and the function burns `nftId` at the end.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


- User calls `mergeTVS(projectId, mergedNftId, [projectId], [mergedNftId])`.
- `_merge()` is executed with `mergedTVS` and `TVSToMerge` pointing to the same allocation.
- Flows are re-pushed into the same allocation, then `nftContract.burn(nftId)` is called, burning `mergedNftId`.
- `claimTokens()` later requires `extOwnerOf(mergedNftId)`, which now reverts, making all flows unclaimable.

### Impact

The user can accidentally lock their own TVS permanently. The associated vesting becomes impossible to claim.

### PoC

_No response_

### Mitigation

Consider adding a simple guard in `mergeTVS()` to reject self-merge inputs, e.g.:

```solidity
for (uint256 i; i < nbOfNFTs; ) {
    require(nftIds[i] != mergedNftId, "Cannot merge NFT into itself");
    // existing logic...
}
  