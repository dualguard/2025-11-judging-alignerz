# [000486] Method AlignerzVesting::splitTVS() calculates split amounts incorrectly
  
  ### Summary

Method `AlignerzVesting::splitTVS()` is used to split one NFT into multiple NFTs and divide the amounts by the percentage provided. But the amounts calculated are incorrect. When `i = 0`, original `splitNftId` is used for calculation. But `_assignAllocation()` overwrites the total allocation. So, from the 2nd iteration the **allocation** used in `_computeSplitArrays(allocation, percentage, nbOfFlows);` will be incorrect.

```solidity
            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
```

### Root Cause

The `allocation` object used is the storage object. So, when `_assignAllocation()` updates the allocation of `splitNftId`, all the next iterations of the loop will use incorrect `allocation` object.

```solidity
   function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
@>      (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
@>          Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
@>          _assignAllocation(newAlloc, alloc);
            allocationOf[nftId] = newAlloc;
            allAlloc[i].amounts = alloc.amounts;
            allAlloc[i].vestingPeriods = alloc.vestingPeriods;
            allAlloc[i].vestingStartTimes = alloc.vestingStartTimes;
            allAlloc[i].claimedSeconds = alloc.claimedSeconds;
            allAlloc[i].claimedFlows = alloc.claimedFlows;
            allAlloc[i].assignedPoolId = alloc.assignedPoolId;
            allAlloc[i].token = alloc.token;
            emit TVSSplit(projectId, isBiddingProject, splitNftId, nftId, allAlloc[i].amounts, allAlloc[i].vestingPeriods, allAlloc[i].vestingStartTimes, allAlloc[i].claimedSeconds);
            unchecked {
                ++i;
            }
        }
        require(sumOfPercentages == BASIS_POINT, Percentages_Do_Not_Add_Up_To_One_Hundred());
        return (splitNftId, newNftIds);
    }

```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088-L1091C13

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Any user who uses `splitTVS()` will end up with fraction of its allocation divided between new NFTs.

### Impact

1. Alice has $100 worth of allocation in original NFT
2. Alice split its NFT for 10 equal parts.
3. But logic of `splitTVS()` will assign $10 to first NFT and $1 each to other 9 NFTs.
4. In total the user will end up with total allocation of $19 spread across 10 NFTs instead of $100.

### PoC

_No response_

### Mitigation

Use `memory` instead of `storage` as below.
```diff
   function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
--      (Allocation storage allocation, IERC20 token) = isBiddingProject ?
++      (Allocation memory allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
            allocationOf[nftId] = newAlloc;
            allAlloc[i].amounts = alloc.amounts;
            allAlloc[i].vestingPeriods = alloc.vestingPeriods;
            allAlloc[i].vestingStartTimes = alloc.vestingStartTimes;
            allAlloc[i].claimedSeconds = alloc.claimedSeconds;
            allAlloc[i].claimedFlows = alloc.claimedFlows;
            allAlloc[i].assignedPoolId = alloc.assignedPoolId;
            allAlloc[i].token = alloc.token;
            emit TVSSplit(projectId, isBiddingProject, splitNftId, nftId, allAlloc[i].amounts, allAlloc[i].vestingPeriods, allAlloc[i].vestingStartTimes, allAlloc[i].claimedSeconds);
            unchecked {
                ++i;
            }
        }
        require(sumOfPercentages == BASIS_POINT, Percentages_Do_Not_Add_Up_To_One_Hundred());
        return (splitNftId, newNftIds);
    }
```
  