# [000994] index overwrite in `rewardProject.kolTVSIndexOf[kol]`
  
  ### Summary

For high participation projects, it will be required to call `setTVSAllocation` more than once since the iteration may hit block gas limit if called once for high number of participants (KOLs). However, `rewardProject.kolTVSIndexOf[kol] = i` in the highlight below will result in giving the same index to more than one kol if called more than once.
```solidity
    function setTVSAllocation(uint256 rewardProjectId, uint256 totalTVSAllocation, uint256 vestingPeriod, address[] calldata kolTVS, uint256[] calldata TVSamounts) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        rewardProject.vestingPeriod = vestingPeriod;
        uint256 length = kolTVS.length;
        require(length == TVSamounts.length, Array_Lengths_Must_Match());
        uint256 totalAmount;
        for (uint256 i = 0; i < length; i++) {
            address kol = kolTVS[i];
            rewardProject.kolTVSAddresses.push(kol);
            uint256 amount = TVSamounts[i];
            rewardProject.kolTVSRewards[kol] = amount;
            //@audit there can be an overwrite here if we call this function in batches
            //for big enough project with huge participant we are calling this in batch
@>>         rewardProject.kolTVSIndexOf[kol] = i; //rewardProject.kolTVSAddresses.length -1;
            totalAmount += amount;
            emit TVSAllocated(rewardProjectId, kol, amount, vestingPeriod);
        }
        require(
            totalTVSAllocation == totalAmount, Amounts_Do_Not_Add_Up_To_Total_Allocation()
        );
        rewardProject.token.safeTransferFrom(msg.sender, address(this), totalTVSAllocation);
    }
```

Impact here will be such that:
Within the claim flow, there is writting of data into the `rewardProject.kolTVSAddresses[index] ` mapping. if since 2 users currently have the same index due to the explanation above, if both users claim, there is a double writing into `rewardProject.kolTVSAddresses[index] ` as highlighted below, and the user whose address was written first gets lost. The user will no longer exist in the array. 
```solidity
    function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        uint256 amount = rewardProject.kolTVSRewards[kol];
        require(amount > 0, Caller_Has_No_TVS_Allocation());
        rewardProject.kolTVSRewards[kol] = 0;
        uint256 nftId = nftContract.mint(kol);
        rewardProject.allocations[nftId].amounts.push(amount);
        uint256 vestingPeriod = rewardProject.vestingPeriod;
        rewardProject.allocations[nftId].vestingPeriods.push(vestingPeriod);
        rewardProject.allocations[nftId].vestingStartTimes.push(rewardProject.startTime);
        rewardProject.allocations[nftId].claimedSeconds.push(0);
        rewardProject.allocations[nftId].claimedFlows.push(false);
        rewardProject.allocations[nftId].token = rewardProject.token;
        allocationOf[nftId] = rewardProject.allocations[nftId];
        uint256 index = rewardProject.kolTVSIndexOf[kol];
        uint256 arrayLength = rewardProject.kolTVSAddresses.length;
        rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
        address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
        rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
@>>     rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
        rewardProject.kolTVSAddresses.pop();
        emit RewardTVSClaimed(rewardProjectId, kol, nftId, amount, vestingPeriod);
    }
``` 

### Root Cause

Wrong indexing

### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

* Owner batches KOL allocations and calls setTVSAllocation multiple times.
* Multiple KOLs are assigned the same kolTVSIndexOf index due to index overwrite.
* First KOL calls claim first, triggering array-swap logic with a shared index.
* Contract overwrites kolTVSAddresses[index] with another KOL's address.
* The original KOL at that index is permanently removed from the array.
* Second KOL with the same index claims and overwrites the same slot again.
* Legitimate KOL permanently loses their array entry and becomes unclaimable.
* Rewards become corrupted, mismatched, or unrecoverable for affected KOLs.

### Impact

Loss of claims

### PoC

_No response_

### Mitigation

```diff
- rewardProject.kolTVSIndexOf[kol] = i;
+ rewardProject.kolTVSIndexOf[kol] = rewardProject.kolTVSAddresses.length -1;
```
  