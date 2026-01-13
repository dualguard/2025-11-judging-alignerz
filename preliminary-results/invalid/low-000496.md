# [000496] Method AlignerzVesting::distributeRemainingStablecoinAllocation() skip element at index 0
  
  ### Summary

The following method will break the loop when `i = 0`, Hence element at index 0 is skipped.

```solidity
    /// @notice Allows the owner to distribute the Stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
@>        for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
            address kol = rewardProject.kolStablecoinAddresses[i];
            uint256 amount = rewardProject.kolStablecoinRewards[kol];
            rewardProject.kolStablecoinRewards[kol] = 0;
            rewardProject.stablecoin.safeTransfer(kol, amount);
            rewardProject.kolStablecoinAddresses.pop();
            emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
            unchecked {
                --i;
            }
        }
    }
```

### Root Cause

The following **for** loop is the root cause.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L585

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

When owner will execute `distributeRemainingStablecoinAllocation()` method, it'll skip element 0.

### Impact

One of the **KoLs** will be left out unintentionally.

### PoC

_No response_

### Mitigation

```diff
    /// @notice Allows the owner to distribute the Stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
++       for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length >= 0;) {
--        for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
            address kol = rewardProject.kolStablecoinAddresses[i];
            uint256 amount = rewardProject.kolStablecoinRewards[kol];
            rewardProject.kolStablecoinRewards[kol] = 0;
            rewardProject.stablecoin.safeTransfer(kol, amount);
            rewardProject.kolStablecoinAddresses.pop();
            emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
            unchecked {
                --i;
            }
        }
    }
```
  