# [000719] Double fee charging when merging previously-merged NFTs
  
  ### Summary

The `mergeTVS` function charges merge fees twice on the same token amounts when merging into an NFT that was previously created through a merge operation. This results in users being charged fees multiple times on the same tokens. 

### Root Cause

In the `mergeTVS` function in `AlignerzVesting.sol`, fees are calculated and deducted from the base NFT (the NFT being merged into) before merging additional NFTs into it. When the base NFT already contains multiple flows from previous merge operations, the function incorrectly charges fees again on those previously-merged amounts.
```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    Allocation storage mergedTVS = ...;
    
    uint256[] memory amounts = mergedTVS.amounts;
    uint256 nbOfFlows = mergedTVS.amounts.length;
    
    // @audit charges fee on ALL flows in mergedNftId, including previously-merged ones
    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
    mergedTVS.amounts = newAmounts;
    
    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }
    
    token.safeTransfer(treasury, feeAmount);
}
```
When an NFT has multiple flows because it was previously merged, those flows already had fees charged during the first merge. Charging fees again on the same flows means users pay merge fees multiple times on the same token amounts.

### Internal Pre-conditions

1. User needs to have performed at least one merge operation previously, creating an NFT with multiple flows.
2. Merge fee rate needs to be set to a non-zero value.
3. User attempts to merge additional NFTs into the previously-merged NFT.

### External Pre-conditions

_No response_

### Attack Path

**Step 1 - First merge (works correctly):**
1. Alice owns NFT 1 with 1 flow of 1000 tokens
2. Alice owns NFT 2 with 1 flow of 1000 tokens  
3. Alice calls `mergeTVS` and merges NFT 2 into NFT 1 with 2% merge fee
4. Fee on the main NFT is charged so NFT 1, only one flow gets charged: 1000 × 2% = 20 tokens fee → 980 tokens remain
5. Fee on the NFTs that will be merged is charged, so NFT 2 gets charged: 1000 × 2% = 20 tokens fee → 980 tokens added to NFT 1
6. NFT 1 now has 2 flows: [980, 980] = 1960 tokens total
7. Total fees paid: 40 tokens (correct)

**Step 2 - Second merge (double charging occurs):**
1. Alice owns NFT 3 with 1 flow of 1000 tokens
2. Alice tries to merge NFT 3 into NFT 1 (which already has 2 flows from previous merge)
3. **First fee charge(on the main NFT):** NFT 1's existing flows get charged again:
   - Flow 0: 980 × 2% = 19.6 tokens
   - Flow 1: 980 × 2% = 19.6 tokens
   - Total: ~39 tokens deducted from NFT 1's existing flows
   - NFT 1 flows reduced to: [960, 960]
4. **Second fee charge(on the NFTs that will be merged):** NFT 3's flow gets charged:
   - 1000 × 2% = 20 tokens
   - NFT 3 flow reduced to 980 tokens and added to NFT 1
5. NFT 1 now has 3 flows: [960, 960, 980] = 2900 tokens total
6. Total fees paid in this merge: ~59 tokens


### Impact

Users who merge already merged NFTs will pay the fee each time, although the fee was paid the first time they merged the NFT. 

### PoC

_No response_

### Mitigation

Remove the fee calculation on the base NFT before merging. Only charge fees on the NFTs being merged in:

```diff
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    address nftOwner = nftContract.extOwnerOf(mergedNftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    
    bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
    (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
    (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

-    uint256[] memory amounts = mergedTVS.amounts;
-    uint256 nbOfFlows = mergedTVS.amounts.length;
-    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
-    mergedTVS.amounts = newAmounts;

    uint256 nbOfNFTs = nftIds.length;
    require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
    require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

    uint256 feeAmount = 0;
    for (uint256 i; i < nbOfNFTs; i++) {
        // Only charge fees on NFTs being merged in
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }
    
    token.safeTransfer(treasury, feeAmount);
    emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
    return mergedNftId;
}
```

This ensures fees are only charged once per flow when it's first merged, preventing double/triple/quadruple charging on subsequent merges.
  