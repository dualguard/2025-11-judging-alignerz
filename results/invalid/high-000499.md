# [000499] Method AlignerzVesting::distributeRewardTVS() is missing onlyOwner modifier
  
  ### Summary

Method `distributeRewardTVS()` can be executed by anyone. Although the comment implies it should be called by the owner only.

```solidity
    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
@>    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
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

### Root Cause

The `onlyOwner` modifier is missing here

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L525C1-L535C6

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

After the `rewardProject.claimDeadline` has passed, any Kol can execute `AlignerzVesting::distributeRewardTVS()` and claim its rewards.

### Impact

Unintended behaviour. Only owner should be able to call `AlignerzVesting::distributeRewardTVS()`.

### PoC

_No response_

### Mitigation

Add the only owner modifier as below:

```diff
++   function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external  onlyOwner {
--    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
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
  