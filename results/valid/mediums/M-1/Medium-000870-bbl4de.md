# [000870] `mergeTVS()` fails to update `allocationOf` for the `mergedNftId`
  
  ### Summary

`mergeTVS()` function updates the per-project storage variables, but fails to update `allocationOf[mergedNftId]` like all the other functions altering the allocation ( `claimNFT()`, `_claimRewardsTVS()` and `splitTVS()` ). 

This causes loss of funds in a form of dividends distribution, for the NFT owners that decided to use merging functionality. 

### Root Cause

`allocationOf` is not updated for the merged nft nor reset for the old ones. 

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

Dividend distribution logic that relies on the global `allocationOf` mapping will see stale values for the merged NFTs. It causes an incorrect dividend weighting, leading to loss of user funds ( other users profit ).

### PoC

_No response_

### Mitigation

Update the `allocationOf` mapping for the `mergedNftId` with the mergedTVS state. 
  