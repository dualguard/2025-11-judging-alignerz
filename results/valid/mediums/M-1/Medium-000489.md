# [000489] Method AlignerzVesting::mergeTVS() doesn't update allocationOf mapping. Which￼ impact A26ZDividendDistributor::getUnclaimedAmounts()
  
  ### Summary

When a user calls `AlignerzVesting::mergeTVS()`, their NFTs `amounts` are merged in single allocation of NFT. But, `allocationOf` mapping is not updated. Hence, if any dividends are distributed, the user will not get dividends of all the underlying NFTs that were merged into one.

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L998-L1048
- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141-L146

### Root Cause

The method `AlignerzVesting::mergeTVS()` doesn't update `allocationOf` mapping.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Any user who `mergeNFTs` won't get the dividend shares of all the underlying NFTs that were merged into single NFT.

### Impact

The users will get less dividend shares even if they are holding a large share or unclaimed tokens. If the unclaimed tokens are from NFTs that were merged to a single one.

### PoC

_No response_

### Mitigation

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
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
++      allocationOf[mergedNftId] = mergedTVS;
        token.safeTransfer(treasury, feeAmount);
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }
```
  