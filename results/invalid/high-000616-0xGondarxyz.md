# [000616] Lack of access control in distributeStablecoinAllocation renders claimDeadline useless
  
  ### Summary

In `AlignerzVesting` contract, a `claimDeadline` is enforced. KOLs should only be able to call `claimStablecoinAllocation` before the deadline, and the owner should be able to call `distributeStablecoinAllocation` after the deadline. But, since there’s no access control on `distributeStablecoinAllocation`, `claimDeadline` becomes completely useless, as a KOL can claim their stale coin allocation at any time, before or after the claim deadline.

Note: A similar issue exists in reward distribution. Since they are both different functions for different functionalities, I submit them as 2 separate issues and they should not be confused.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L537

### Details

See that `claimStablecoinAllocation` function can be called by KOLs before the claimDeadline

```solidity
    /// @notice Allows a KOL to claim his stablecoin allocation
    /// @param rewardProjectId Id of the rewardProject
    function claimStablecoinAllocation(uint256 rewardProjectId) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
     =>   require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());
        address kol = msg.sender;
        _claimStablecoinAllocation(rewardProjectId, kol);
    }
```

But there is another function, `distributeStablecoinAllocation` , that should be called by the owner to distribute the stablecoins that have not been yet claimed to the KOLs

```solidity
=>    /// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
   =>     require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

But this function has no access control, which means, not only the owner, but anyone, including KOLs can call this function.

Which also means KOLs can claim their stablecoins

- before the claimDeadline, by calling claimStablecoinAllocation
- and also after the claimDeadline, by calling distributeStablecoinAllocation

Which also means that the enforcement of claimDeadline is completely non functional.

This is obviously not intended by the dev team, as mentione in the comment above `distributeRewardTVS` 

    */// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs*

and breaks the concept of claimDeadline.

### Impact: High
Complete bypass of core access control mechanism. The claimDeadline - a fundamental protocol feature - becomes entirely non-functional. Any KOL can claim rewards at any time by exploiting the missing access control, completely breaking the intended two-phase distribution model and removing owner control over post-deadline distributions.

Likelihood: High

 Trivially exploitable - requires only a single function call. No special conditions needed.

Final Severity: High

### Recommendation

Add access control on `distributeStablecoinAllocation`
  