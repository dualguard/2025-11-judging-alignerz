# [000764] missing `onlyOwner` modifiers on the reward distribution functions
  
  ### Summary

The missing `onlyOwner` modifiers on the reward distribution functions will cause unauthorized triggering of reward and stablecoin distributions for KOLs as any external user can execute these admin-intended flows on behalf of the protocol.

###  Root Cause

In `AlignerzVesting.sol:522-550` the functions `distributeRewardTVS` and `distributeStablecoinAllocation` are documented as owner-only operations but are declared `external` without `onlyOwner`, so any address can call them once the claim deadline has passed.

```solidity
    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
>>  function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
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

    /// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
>>  function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external { 
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

The intent in the NatSpec comments is that only the owner should decide when to bulk-distribute unclaimed TVS and stablecoin rewards after the claim window, but because the `onlyOwner` modifier is missing on both functions, any external caller can trigger `_claimRewardTVS` and `_claimStablecoinAllocation` for the specified addresses as soon as `block.timestamp > claimDeadline`.

###  Internal Pre-conditions

1. A reward project must exist and be configured with TVS and stablecoin allocations to some KOL addresses via `setTVSAllocation` and `setStablecoinAllocation`.  
2. The reward project’s `claimDeadline` must have passed (`block.timestamp > rewardProject.claimDeadline`) so that the distribution functions no longer revert on the deadline check.

### External Pre-conditions

1. No external oracle or third-party protocol condition is required; only the passage of time beyond `claimDeadline` is needed.  
2. An arbitrary attacker address must be able to send transactions to the vesting contract (normal on any public EVM chain).

###  Attack Path

1. The owner creates a reward project with `launchRewardProject`, then sets KOL allocations via `setTVSAllocation` and `setStablecoinAllocation`, transferring both TVS and stablecoin into the vesting contract.  
2. Some KOLs do not manually claim during the claim window, so their allocations remain in the contract past `claimDeadline`.  
3. After `block.timestamp > claimDeadline`, an external attacker address (not equal to the vesting owner) prepares arrays `kolTVS` and `kolStable` that match the KOL addresses recorded in `rewardProject.kolTVSAddresses` and `rewardProject.kolStablecoinAddresses`.  
4. The attacker calls `distributeRewardTVS(rewardProjectId, kolTVS)` from their own address; because there is no `onlyOwner`, the call succeeds and `_claimRewardTVS` is executed for each listed KOL, minting NFTs and zeroing their TVS rewards.  
5. The attacker then calls `distributeStablecoinAllocation(rewardProjectId, kolStable)` from the same non-owner address; again, the lack of `onlyOwner` allows this to succeed, transferring stablecoin rewards from the vesting contract to the KOL addresses.  
6. The net result is that non-admin users can decide when these admin-intended distributions are executed, taking this control away from the protocol owner (although they do not redirect funds to themselves).

### Impact

The protocol owner loses exclusive control over when post-deadline bulk reward and stablecoin distributions are executed, since any user can trigger `distributeRewardTVS` and `distributeStablecoinAllocation` after the claim window, leading to unplanned or premature distribution timing that is not authorized by the admin (griefing of admin control, but not direct theft).

### PoC


###  Mitigation

Add the `onlyOwner` modifier to both functions so that only the protocol admin can initiate bulk reward and stablecoin distributions, aligning the implementation with the documented intent.
  