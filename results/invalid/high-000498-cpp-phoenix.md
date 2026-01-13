# [000498] Missing onlyOwner check in AlignerzVesting::distributeStablecoinAllocation()
  
  ### Summary

Any **Kol** can call `AlignerzVesting::distributeStablecoinAllocation()` and claim their stablecoin allocation post `rewardProject.claimDeadline`.
```solidity
@>    /// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
@>    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

### Root Cause

Missing access control

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Method `AlignerzVesting::distributeStablecoinAllocation()` can be called by anyone.

### Impact

Because of missing `onlyOwner` modifier check, any **Kol** can call `AlignerzVesting::distributeStablecoinAllocation()`.

### PoC

_No response_

### Mitigation

Add `onlyOwner` access control as below:
```diff
++   function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external onlyOwner {
--    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```
  