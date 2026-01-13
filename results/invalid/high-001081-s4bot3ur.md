# [001081] Missing onlyOwner Modifier Allows Anyone to Distribute Unclaimed KOL Rewards
  
  ### Summary

The missing onlyOwner modifier in `distributeRewardTVS()` and `distributeStablecoinAllocation()` will cause unauthorized distribution of unclaimed KOL rewards as any external user can trigger reward distribution after the claim window.

This results in a loss of administrative control for the protocol owner, as anyone can force-call these operations, triggering reward claims after deadline.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540

there is no `onlyOwner` modifier, even though these functions are clearly intended to be admin-only distribution functions.

This missing access control allows any external user to trigger `_claimRewardTVS` or `_claimStablecoinAllocation` for arbitrary KOL addresses.

### Internal Pre-conditions

1. The reward project must have KOL allocations stored in:
    - rewardProjects[rewardProjectId].kolTVSRewards
    - or rewardProjects[rewardProjectId].kolStablecoinRewards
2. The claimDeadline must have passed, allowing Only to owner to distribute.
3. The KOL reward lists (kolTVSAddresses or kolStablecoinAddresses) must contain at least one unclaimed entry.

### External Pre-conditions

1. An attacker simply needs to call `AlignerzVesting::distributeRewardTVS` or `AlignerzVesting::distributeStablecoinAllocation` passing the reward project id and KOL address.


### Attack Path

1. Attacker waits until the reward project's `claimDeadline` has passed.
2. Attacker calls `distributeRewardTVS(projectId, attackerProvidedKolArray)`      
providing any valid `kol[]` array.
3. `_claimRewardTVS` executes for each provided KOL, minting NFTs or transferring allocations.
4. Alternatively, attacker calls:
`distributeStablecoinAllocation(projectId, kol[])` to force stablecoin distributions.

### Impact

The protocol loses administrative control over KOL reward distribution. Although no direct theft occurs, the ability to trigger and finalize reward execution is taken away from the owner, resulting in a critical governance failure. This effectively becomes an admin-privilege escalation vulnerability that impacts all current and future reward projects.

### PoC

_No response_

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
    .

    /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
-   function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
+   function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external onlyOwner{
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
-    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
+    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external onlyOwner{
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

    .
    .
    .

}
```
  