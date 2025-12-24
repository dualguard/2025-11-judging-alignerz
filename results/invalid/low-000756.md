# [000756] `distributeRemainingStablecoinAllocation` does zero amount `safeTransfer` calls that can brick distribution with non standard stablecoins
  
  ### Summary

The remaining-stablecoin distribution loop blindly calls `rewardProject.stablecoin.safeTransfer(kol, amount)` even when `amount` is zero, so if a non-standard stablecoin (that reverts on zero transfers) is used, the entire distribution can revert and get stuck for all KOLs.

### Root Cause

In `AlignerzVesting.sol:581-591` the function does not enforce that each `rewardProject.kolStablecoinRewards[kol]` is strictly positive before attempting the transfer.

```solidity
function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolStablecoinAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
        address kol = rewardProject.kolStablecoinAddresses[i];
        uint256 amount = rewardProject.kolStablecoinRewards[kol];
        rewardProject.kolStablecoinRewards[kol] = 0;
>>    rewardProject.stablecoin.safeTransfer(kol, amount); 
        rewardProject.kolStablecoinAddresses.pop();
        emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
        unchecked {
            --i;
        }
    }
}
```

When `kolStablecoinRewards[kol]` entries are zero , and the configured stablecoin i reverts on zero-amount transfers, the first zero-amount iteration will revert and halt the entire loop, preventing remaining positive allocations from being distributed.

### Impact

Using a non-standard stablecoin and/or zero allocations can cause `distributeRemainingStablecoinAllocation` to revert and lock remaining undistributed stablecoin in the contract until the bug is fixed.

### Mitigation

Add a simple guard to skip zero-amount transfers:

```diff
uint256 amount = rewardProject.kolStablecoinRewards[kol];
rewardProject.kolStablecoinRewards[kol] = 0;
+ if (amount > 0) {
+    rewardProject.stablecoin.safeTransfer(kol, amount);
+ }
```
  