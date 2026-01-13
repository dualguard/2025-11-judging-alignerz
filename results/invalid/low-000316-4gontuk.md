# [000316] Cleanup reward distribution functions revert when no remaining KOLs in `AlignerzVesting`
  
  ### Description:
In `AlignerzVesting`, the owner-only cleanup functions [`distributeRemainingRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L554-L577) and [`distributeRemainingStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L581-L596) are meant to distribute any unclaimed rewards after the `claimDeadline`. Both functions iterate over the corresponding KOL address arrays from the end:

```solidity
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
        address kol = rewardProject.kolTVSAddresses[i];
        uint256 amount = rewardProject.kolTVSRewards[kol];
        rewardProject.kolTVSRewards[kol] = 0;
        uint256 nftId = nftContract.mint(kol);
        rewardProject.allocations[nftId].amounts.push(amount);
        // ...
        rewardProject.kolTVSAddresses.pop();
        unchecked { --i; }
    }
}
```

The loop index `i` is initialized as `len - 1`. When there are no remaining KOLs (`len == 0`), this becomes `0 - 1`, which underflows in Solidity `0.8.x` and causes an immediate revert before the loop condition is evaluated. The same pattern is present in `distributeRemainingStablecoinAllocation()` using `rewardProject.kolStablecoinAddresses`. As a result, these “remaining distribution” functions revert in the benign case where there is nothing left to distribute, preventing them from being used as safe, idempotent sweep functions by the owner or off-chain tooling.

### Impact:
Owner-only cleanup functions revert when called after all KOLs have already claimed, breaking idempotent “sweep remaining rewards” calls and slightly degrading admin UX.

### Recommendation:
Guard against the empty-case before computing `len - 1`, or restructure the loop to avoid underflow. For example:

```solidity
uint256 len = rewardProject.kolTVSAddresses.length;
if (len == 0) return;
for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
    // existing logic
}

uint256 len = rewardProject.kolStablecoinAddresses.length;
if (len == 0) return;
for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
    // existing logic
}
```

Alternatively, iterate using a decreasing index that is safe for zero length:

```solidity
for (uint256 i = rewardProject.kolTVSAddresses.length; i > 0;) {
    i--;
    address kol = rewardProject.kolTVSAddresses[i];
    // logic...
    rewardProject.kolTVSAddresses.pop();
}
```

Apply the same pattern to `distributeRemainingStablecoinAllocation()`.

  