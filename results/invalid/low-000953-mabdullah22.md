# [000953] Reward Project Can Be Created With Already-Expired Claim Window
  
  ### Summary

The `launchRewardProject` function does not validate that the `startTime` is in the future or that the calculated `claimDeadline` is after the current block timestamp. This allows the owner to create reward projects with claim windows that are partially or fully in the past, making it impossible for KOLs to claim their rewards during the intended claim period.


### Root Cause

The root cause is in [`launchRewardProject`](AlignerzVesting.sol) 

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
    rewardProject.startTime = startTime;
    rewardProject.claimDeadline = startTime + claimWindow;  // No validation
    rewardProject.token = IERC20(tokenAddress);
    rewardProject.stablecoin = IERC20(stablecoinAddress);

    emit RewardProjectLaunched(rewardProjectCount, tokenAddress);
    rewardProjectCount++;
}
```

Missing validations:
1. No check that `startTime >= block.timestamp`
2. No check that `claimWindow > 0`
3. No check that `claimDeadline > block.timestamp`

### Internal Pre-conditions

1. Owner must call `launchRewardProject` with:
   - `startTime` in the past, OR
   - `startTime + claimWindow <= block.timestamp`

### External Pre-conditions

_No response_

### Attack Path


**Scenario 1: Past startTime**
1. Current time: `1000`
2. Owner calls `launchRewardProject` with:
   - `startTime = 500` (in the past)
   - `claimWindow = 100`
   - `claimDeadline = 600` (in the past)
3. Owner sets allocations via `setTVSAllocation`
4. KOLs attempt to claim via `claimRewardTVS`
5. Transaction reverts: `require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed())`

**Scenario 2: Insufficient claimWindow**
1. Current time: `1000`
2. Owner calls `launchRewardProject` with:
   - `startTime = 1000`
   - `claimWindow = 10` (very short)
   - `claimDeadline = 1010`
3. By the time allocations are set and KOLs are notified, time is `1015`
4. All claims revert due to expired deadline


### Impact

1. **Unusable projects**: Reward projects become non-functional if claim window is expired
2. **Locked funds**: Tokens transferred to the contract cannot be claimed by KOLs
3. **Admin burden**: Owner must use `distributeRemainingRewardTVS` to manually distribute all rewards
4. **User experience**: KOLs cannot self-claim, reducing autonomy
5. **Configuration errors**: Easy to make mistakes with timestamp parameters

### PoC

_No response_

### Mitigation

Add timestamp validations to `launchRewardProject`.
  