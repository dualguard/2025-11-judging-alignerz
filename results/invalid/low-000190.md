# [000190] `distributeStablecoinAllocation`  and `distributeRewardTVS` functions emit the wrong event.
  
  ### Summary

The `distributeStablecoinAllocation` function is meant to distribute the stablecoin allocation to a given array of addresses who didn't claim it before the claim deadline. It is defined as follows:

```solidity
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            // @audit LOW REPORTED: wrongly emits StablecoinAllocationClaimed instead of StablecoinAllocationsDistributed
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

On the other hand,  `distributeRemainingStablecoinAllocation` function is an `onlyOwner` function that allows to distribute the stablecoin allocation to all addresses who didn't claim before the claim deadline. This function emits a `StablecoinAllocationsDistributed` event.

The problem arises because contrary to `distributeRemainingStablecoinAllocation` function, `distributeStablecoinAllocation` doesn't emit a `StablecoinAllocationsDistributed` event, but a `StablecoinAllocationClaimed` event instead.

The issue is the same with `distributeRewardTVS` function which allow to distribute the TVS to a given array of addresses:

```solidity
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i; i < len;) {
            // @audit LOW REPORTED: wrongly emits RewardTVSClaimed instead of RewardTVSDistributed
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

It emits  `RewardTVSClaimed` events in `_claimRewardTVS` instead of `RewardTVSDistributed` events.

### Root Cause

The root cause of this issue is that:
- `distributeStablecoinAllocation` function uses internally  `_claimStablecoinAllocation` which is incorrect, because this internal function will emit a `StablecoinAllocationClaimed` event.
- `distributeRewardTVS` function uses internally `_claimRewardTVS` which is incorrect, because this internal function will emit a `RewardTVSClaimed` event.

But in the case of distributions, the claim period is over, so this is not a claim, it is a distribution.

### Impact

The impact of this issue is low as it results in a wrong event emission which may lead to off-chain monitoring issues.

### Mitigation

Make sure to emit the correct event in the case of `distributeStablecoinAllocation` and `distributeRewardTVS` functions. The code probably needs to be refactored.
  