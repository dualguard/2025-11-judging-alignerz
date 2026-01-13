# [000321] Public post-deadline reward distribution functions contradict “owner-only” specification in `AlignerzVesting`
  
  ### Description:
The functions [`distributeRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535) and [`distributeStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550) in `AlignerzVesting` are declared as plain `external` functions without the `onlyOwner` modifier but are documented as “Allows the owner to distribute…”. This creates a mismatch between the intended access control and the actual behavior, allowing any address to trigger post-deadline distribution of unclaimed KOL rewards.

For example:

```solidity
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked { ++i; }
    }
}
```

Since `_claimRewardTVS()` and `_claimStablecoinAllocation()` always use the stored allocations and recipient addresses, an arbitrary caller cannot redirect or steal funds. However, they can:
- Force post-deadline distribution earlier than the owner may intend.
- Cause unnecessary reverts by passing a `kol` array shorter than `rewardProject.kolTVSAddresses.length` or `rewardProject.kolStablecoinAddresses.length`, resulting in out-of-bounds access and wasted gas.

This inconsistent access control is also contrasted by the related `distributeRemainingRewardTVS()` and `distributeRemainingStablecoinAllocation()` functions, which are explicitly restricted with `onlyOwner`.

### Impact:
Low-severity access-control / specification mismatch enabling permissionless triggering and minor griefing of post-deadline reward distributions.

### Recommendation:
Either:
- Enforce the documented behavior by adding the `onlyOwner` modifier to `distributeRewardTVS()` and `distributeStablecoinAllocation()`, and ensure they iterate using the contract’s internal KOL arrays; or
- If permissionless triggering is intended, update the NatSpec comments to remove the “owner” wording and adjust the loops to use `kol.length` (with appropriate checks) to avoid unexpected reverts and clarify the intended API semantics.

  