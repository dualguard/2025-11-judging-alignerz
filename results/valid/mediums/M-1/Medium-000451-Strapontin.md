# [000451] Merging NFTs will not update the `allocationOf` of the NFT that is being merged into
  
  ### Summary

The project allows NFTs to be split or merged at will by users. While splitting an NFT will correctly update the `allocationOf` of all created NFTs, merging does not update this variable. The issue with this is that `allocationOf` is critical to determine the amounts of dividends that should be claimed to users. When a user merges 2 NFTs together, they will only be able to claim the dividends for the NFT allocation value that was before the merge, effectivelly losing the dividends for the merged NFTs.

### Root Cause

`mergeTVS` does not update `allocationOf` for the remaining NFT.

### Internal Pre-conditions

2 NFTs are created.

### Attack Path

1. Alice owns 2 NFTs for one project. `allocationOf[nft1].amount = 1e6` and `allocationOf[nft2].amount = 100e6` 
2. Alice merges her NFTs in order to have only 1 for the project. Assuming the fees are 0, merging nft2 into nft1 does not update any `allocationOf`
3. When dividends are distributing, Alice expects to have her amount calculated from 101e6, but it will instead be calculated from 1e6

### Impact

Loss of funds

### Mitigation

Update the merge to correctly update `allocationOf` so dividends can be send correctly.
  