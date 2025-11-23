# [001023] Missing `onlyOwner` modifier allow KOLs to claim the rewards after `claimDeadline` expired
  
  ### Summary

Unrestricted `distributeRewardTVS()` and `distributeStablecoinAllocation()` allow anyone to call them and distribute the rewards to KOLs even after `rewardProject.claimDeadline` expired. 

### Root Cause

Functions [distributeRewardTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) and [distributeStablecoinAllocation()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540) are missing the `onlyOwner` modifier. 
```solidity
    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
//....
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
```


### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Anyone can call mentioned functions to distribute the rewards to late KOLs.

### Impact

Only admin action can be executed by anyone. 

### PoC

_No response_

### Mitigation

Since `AlignerzVesting` contract  implements same admin-only feature in 
`distributeRemainingRewardTVS()` and `distributeRemainingStablecoinAllocation()` functions, consider removing the vulnerable ones:
`distributeRewardTVS()` and `distributeStablecoinAllocation()`
  