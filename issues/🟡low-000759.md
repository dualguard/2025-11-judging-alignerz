# [000759] Missing validation on `startTime` and `claimWindow` in `launchRewardProject` can make a reward project instantly expired
  
  ### Summary

The missing validation on `startTime` and `claimWindow` in `launchRewardProject` can make a reward project instantly expired, so KOLs cannot claim their TVS or stablecoin allocations if the owner misconfigures these values.

### Root Cause

In `AlignerzVesting.sol:433-448` the function accepts any `startTime` and `claimWindow` and just does `rewardProject.startTime = startTime;` and `rewardProject.claimDeadline = startTime + claimWindow;` without checking that `startTime >= block.timestamp` and `claimDeadline > block.timestamp`.

```solidity
    function launchRewardProject(
        address tokenAddress,
        address stablecoinAddress,
        uint256 startTime,
        uint256 claimWindow
    ) external onlyOwner {
        require(tokenAddress != address(0), Zero_Address());
        require(stablecoinAddress != address(0), Zero_Address());

        RewardProject storage rewardProject = rewardProjects[rewardProjectCount];
>>    rewardProject.startTime = startTime; 
>>    rewardProject.claimDeadline = startTime + claimWindow;
        rewardProject.token = IERC20(tokenAddress);
        rewardProject.stablecoin = IERC20(stablecoinAddress);

        emit RewardProjectLaunched(rewardProjectCount, tokenAddress);
        rewardProjectCount++;
    }
```

### Impact 

A wrong configuration (past `startTime`, zero/small `claimWindow`) can make `block.timestamp < rewardProject.claimDeadline` false from the start, so `claimRewardTVS` and `claimStablecoinAllocation` will always revert for that project.

### Mitigation

Add simple checks in `launchRewardProject` like `require(startTime >= block.timestamp, Starttime_Must_Be_In_The_Future());` and `require(startTime + claimWindow > block.timestamp, Deadline_Has_Passed());` and `require(claimWindow > 0, Zero_Value());` before setting `startTime` and `claimDeadline`.
  