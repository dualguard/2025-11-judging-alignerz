# [000490] Merge fees are also applied to claimed tokens in AlignerzVesting::mergeTVS()
  
  ### Summary

Method `AlignerzVesting::mergeTVS()` is used to merge multiple NFTs into a single one. We apply `mergeFeeRate` on each NFT separately. But the calculation also applies fees on the amounts that has already been claimed. This will create a deficit of tokens in the `AlignerzVesting.sol` contract. Hence, some users won't be able to claim their tokens.

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013C60-L1013C93
- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1039
- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1023

### Root Cause

The following 2 lines where fees are calculated are the root cause. The protocol is applying fees on the flows that are fully or partially claimed. Hence, the fees deduced are instead partly taken from the allocation of other users.

```solidity
    /// @notice Allows users to merge one TVS into another
    /// @param projectId ID of the biddingProject
    /// @param nftIds tokenIds of the NFTs that will be merged, these tokens will be burned
    /// @param mergedNftId tokenId of the NFT that will remain after the merge
    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        
        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
@>        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
@>        token.safeTransfer(treasury, feeAmount);
        emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
        return mergedNftId;
    }

    function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
        require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());
        
        bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
        (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge ?
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        require(address(token) == address(tokenToMerge), Different_Tokens());

        uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
@>            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
            feeAmount += fee;
        }
        nftContract.burn(nftId);
    }
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Alice has vesting of $100. From which Alice has claimed $50.
2. Alice wants to merge an NFT. Now, the fees will be calculated on a total of $100. 
3. Let's say the fee is 10%. So, 10% of $100 = $10. Which will be taken and sent to the treasury.
4. Now, Alice's total amount is $90. But Alice has claimed 50% of it. So, Alice can claim 50% of $90 = $45.
5. But, the protocol took out $10. While Alice only paid $5.
6. Rest $5 was taken from the allocation of others. 
7. So, Some people won't be able to claim their vested tokens because of deficit in the `AlignerzVesting` contract.

### Impact

Some people won't be able to claim their vested tokens because of deficit in the `AlignerzVesting` contract.

### PoC

_No response_

### Mitigation

Only apply merge fees unclaimed tokens. All the claimed tokens and claimed flows should be exempted from the fees. Currently, fees are taken incorrectly.
  