# [000402] Missing validation for startTime and endTime
  
  **Summary:** The `launchRewardProject()` function does not validate whether the `startTime` and `endTime` are in the future relative to the current time.

```solidity
function launchRewardProject(...) external onlyOwner {
...
        RewardProject storage rewardProject = rewardProjects[rewardProjectCount];
        rewardProject.startTime = startTime;
        rewardProject.claimDeadline = startTime + claimWindow;
        rewardProject.token = IERC20(tokenAddress);
        rewardProject.stablecoin = IERC20(stablecoinAddress);
...
    }
```

This can lead to the following issue:

Invalid Launchpad Timing: If `startTime` or `endTime` is in the past, the token sale may start or end immediately.

**Recommendation:** Add validation to ensure that both `startTime` and `endTime` are in the future relative to the current time. Retrieve the current timestamp and compare it with `startTime` and `endTime`.
  