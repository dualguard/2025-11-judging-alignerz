# [000389] Self-Merge in mergeTVS() Burns Target NFT & Permanently Locks Funds
  
  ### Summary

The `mergeTVS()` function does not validate that `mergedNftId` is different from every `nftId` inside nftIds[].
If the caller includes the same NFT:

```solidity
mergeTVS(projectId, mergedNftId = X, nftIds = [X], projectIds = [projectId]);
```

Then:
- _merge() is executed on the same NFT
- _merge() burns the NFT

The storage allocation mapped to that NFT remains, but ownerOf(NFT) now returns zero, User loses access to their entire vesting allocation, Protocol enters a corrupted orphaned-state


### Root Cause

No check that `nftId != mergedNftId`. Neither `mergeTVS()` nor `_merge()` verify that the NFT being merged into is different from the NFT being merged from.



### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

User mistakenly merges an NFT into itself. 
1. user call mergeTVS() with nftId == mergedNftId (12)
2. Ownership check passes (user owns NFT 12).
3. mergedTVS = allocation[12] is loaded.
4. _merge() called with nftId = 12.
5. _merge():  
   - Adds new flow copies to the same array (self-copy)
   - Calculates and charges fee
   - calls burn(12)
6. NFT 12 is burned.
7. User loses their only reference to the vesting allocation.
8. Allocation storage for TVS remains but cannot be claimed.
9. All tokens represented by that TVS are permanently locked.

### Impact

1. Permanent loss of user funds
All vesting tokens locked behind the burned TVS become unclaimable.

2. Orphaned allocation storage
Allocation remains inside but no NFT exists to reference it.
 ```solidity
 biddingProjects[projectId].allocations[nftId]
 ```

### PoC

_No response_

### Mitigation

Add explicit check in both mergeTVS() and _merge():
```solidity
require(nftId != mergedNftId, "Cannot merge NFT into itself");
```

Also add safety in _merge():
 ```solidity
 require(nftId != mergedTVSId, Self_Merge_Not_Allowed());
 ```
  