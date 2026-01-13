# [000974] 🟠 Medium distributeRewwardTVS only works if no KOLs have claimed their reward
  
  ### Summary

The function takes a kol array as an input but iterates over the entire length of kolTVSAddresses for a given project. This means that the array the admin must enter has to be the same length or it will revert. The implication is that the admin must input every single kol for the distribution. However, if that kol has already claimed, their distribution amount will be zero which causes _claimRewardTVS to revert.

### Root Cause

```solidity
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

```solidity
    function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        uint256 amount = rewardProject.kolTVSRewards[kol];
        require(amount > 0, Caller_Has_No_TVS_Allocation()); // <-- if the user doesn't exist or has already claimed, it reverts
        rewardProject.kolTVSRewards[kol] = 0;
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

NA

### Impact

KOLs who have not claimed cannot be distributed to via the admin.

### PoC

_No response_

### Mitigation

_No response_
  