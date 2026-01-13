# [000832] The `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` functions don't have the `onlyOwner` modifier
  
  ### Summary

The `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` functions should be called only by the owner, as stated in the comments above the functions, but they have no access control and can be called by anyone.

### Root Cause

The [`AlignerzVesting::distributeRewardTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) and [`AlignerzVesting::distributeStablecoinAllocation`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540) functions have no access control:

```solidity

/// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
/// @param rewardProjectId Id of the rewardProject
/// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
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

/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
/// @param rewardProjectId Id of the rewardProject
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
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

The comments above state that these functions should be called by the owner, but they actually can be called by anyone.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Anyone can call the `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` functions and distribute rewards.

### Impact

Anyone can call the `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` functions. There is no loss of funds, so the impact of that is Low.

### PoC

_No response_

### Mitigation

Add `onlyOwner` modifier to the `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` functions.
  