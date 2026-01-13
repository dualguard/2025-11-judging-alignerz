# [000531] splitTVS() destroys original NFT allocation data
  
  ## Summary

Original allocation permanently overwritten on first iteration of `splitTVS()`. 



## Root cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107

```solidity
function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
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
@=>            _assignAllocation(newAlloc, alloc);
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

Both `allocation` and `newAlloc` point to the same storage slot: `biddingProjects[projectId].allocations[splitNftId]`/`rewardProjects[projectId].allocations[nftId]`. When `_assignAllocation()` is called, it permanently modifies the original allocation.



## Impact

This vulnerability has critical consequences. It leads to permanent loss of funds by overwriting the `amounts` filed in the `allocation` with amounts that have decreased in percentage.


## Mitigation

```solidity
function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        
+        Allocation memory originalAllocation = Allocation({
+            amounts: allocation.amounts,
+            vestingPeriods: allocation.vestingPeriods,
+            vestingStartTimes: allocation.vestingStartTimes,
+            claimedSeconds: allocation.claimedSeconds,
+            claimedFlows: allocation.claimedFlows,
+            assignedPoolId: allocation.assignedPoolId,
+            token: allocation.token
+        });
    
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
-        allocation.amounts = newAmounts;
        
+        Allocation memory baseAllocation = originalAllocation;
+    	baseAllocation.amounts = newAmounts;
        
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
-            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
+            Allocation memory alloc = _computeSplitArrays(baseAllocation, percentage, nbOfFlows);
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

  