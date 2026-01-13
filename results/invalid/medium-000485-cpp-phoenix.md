# [000485] Method mergeTVS() burns user NFT if mergedNftId is one of nftIds
  
  ### Summary

Method `mergeTVS()` is used to merge mutliple NFTs into one. But there is check missing that checks `mergedNftId !=nftIds`. So, if user provides same nftId in nftIds[] then it's original NFT will be burned too and it'll lose it's allocation or rewards.

```solidity
    function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
        require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());
        
        bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
        (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        require(address(token) == address(tokenToMerge), Different_Tokens());

        uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
            feeAmount += fee;
        }
@>        nftContract.burn(nftId);
    }
```

### Root Cause

The below method burns NFT without validating if `nftId` is equal to `mergedNftId`

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1028C1-L1048C6

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Alice provides mergeNFTId as 1 and nftIds as [2,3,4,5,6,7,8,9,1].
2. Alice is left with no NFT at the end of the execution as her original NFT is also burned.

### Impact

The user can lose it'll 100% of allocation if it provides same NFT id as mergeNFTId in nftIds list

### PoC

_No response_

### Mitigation

Add the following check:

```diff
    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        
        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        //@audit What if the allocation is already claimed???
        //@audit-issue Charging fee on gone period. How???
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
++          if(nftIds[i] == mergedNftId) continue;
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
        token.safeTransfer(treasury, feeAmount);
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
```
  