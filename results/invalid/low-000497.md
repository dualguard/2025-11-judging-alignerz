# [000497] Method AlignerzVesting::distributeRemainingRewardTVS() excludes element 0 while iterating
  
  ### Summary

The below method `AlignerzVesting::distributeRemainingRewardTVS()` should include `i = 0` in the for loop. Otherwise, this method will unintentionally skip the first element in the `rewardProject.kolTVSAddresses` array.

```solidity
    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
@>        for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
            address kol = rewardProject.kolTVSAddresses[i];
            uint256 amount = rewardProject.kolTVSRewards[kol];
            rewardProject.kolTVSRewards[kol] = 0;
            uint256 nftId = nftContract.mint(kol);
            rewardProject.allocations[nftId].amounts.push(amount);
            uint256 vestingPeriod = rewardProject.vestingPeriod;
            rewardProject.allocations[nftId].vestingPeriods.push(vestingPeriod);
            rewardProject.allocations[nftId].vestingStartTimes.push(rewardProject.startTime);
            rewardProject.allocations[nftId].claimedSeconds.push(0);
            rewardProject.allocations[nftId].claimedFlows.push(false);
            rewardProject.allocations[nftId].token = rewardProject.token;
            rewardProject.kolTVSAddresses.pop();
            allocationOf[nftId] = rewardProject.allocations[nftId];
            emit RewardTVSDistributed(rewardProjectId, kol, nftId, amount, vestingPeriod);
            unchecked {
                --i;
            }
        }
    }
```

### Root Cause

Missing `i = 0` edge case 

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L552C5-L577C6

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

When the **owner** executes `AlignerzVesting::distributeRemainingRewardTVS()`, the element `i = 0` is skipped in the for loop.

### Impact

One of the element will be unintentionally skipped. That user will not get his NFT.

### PoC

_No response_

### Mitigation

Update the following method as below:
```diff
    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
++       for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length >= 0;) {
--        for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
            address kol = rewardProject.kolTVSAddresses[i];
            uint256 amount = rewardProject.kolTVSRewards[kol];
            rewardProject.kolTVSRewards[kol] = 0;
            uint256 nftId = nftContract.mint(kol);
            rewardProject.allocations[nftId].amounts.push(amount);
            uint256 vestingPeriod = rewardProject.vestingPeriod;
            rewardProject.allocations[nftId].vestingPeriods.push(vestingPeriod);
            rewardProject.allocations[nftId].vestingStartTimes.push(rewardProject.startTime);
            rewardProject.allocations[nftId].claimedSeconds.push(0);
            rewardProject.allocations[nftId].claimedFlows.push(false);
            rewardProject.allocations[nftId].token = rewardProject.token;
            rewardProject.kolTVSAddresses.pop();
            allocationOf[nftId] = rewardProject.allocations[nftId];
            emit RewardTVSDistributed(rewardProjectId, kol, nftId, amount, vestingPeriod);
            unchecked {
                --i;
            }
        }
    }
```
  