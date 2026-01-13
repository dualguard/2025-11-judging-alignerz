# [000174] Splitting a TVS from a bidding project will result in 1 bidding project TVS and others will be considered reward project.
  
  ### Summary

The `splitTVS` function allows a user to split a TVS (NFT) into multiple, given a certain percentage for each.
The problem is that all newly minted NFT will internally belong to reward projects.

### Root Cause

The issue lies in the `for` loop in `splitTVs` function:

```solidity
            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            // @audit LOW: biddingProject will be split into 1 biddingProject and others rewardProject
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
            allocationOf[nftId] = newAlloc;

```

During the first iteration, `i == 0` and `nftId` will be `splitNftId`. If `splitNftId` belongs to a bidding project, then the `newAlloc` storage `Allocation` will point to the `biddingProjects` storage mapping.

But during all others iterations, a new NFT Is minted and there is no code to check whether it should belong to a bidding project or a reward project. `NFTBelongsToBiddingProject` mapping is never set when the new minted NFT  should belong to a bidding project.
Therefore,  the line `NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;` will always return false after the first iteration, and the info related to this new NFT will be stored in `rewardProjects` mapping.

### Attack Path

There is no specific attack path. Whenever a user splits his bidding project NFT, all the newly minted NFT will have their information stored in the reward project mapping.

### Impact

The impact of this issue is low as it results in an incorrect internal state, but without impacting users funds.

### Mitigation

When a new NFT is minted, check whether the original NFT is from a bidding project or not and update `NFTBelongsToBiddingProject` mapping accordingly.
  