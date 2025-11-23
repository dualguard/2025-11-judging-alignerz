# [000795] Overwriting KOL Allocations Causes Previously Funded Rewards to Become Permanently Stuck
  
  ### Summary

The lack of validation against overwriting existing KOL reward allocations will cause a loss of previously funded rewards for KOLs as the owner can reset allocations without accounting for prior balances, resulting in stuck tokens inside the contract.

### Root Cause

In `AlignerzVesting.sol: setTVSAllocation` and `setStablecoinAllocation`, the contract overwrites `kolTVSRewards[kol]` / `kolStablecoinRewards[kol]` without first clearing or redistributing previously allocated rewards, causing any earlier transferred tokens to become inaccessible.

[Overwriting of kolTVSRewards](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L468)

[Overwriting of kolStablecoinRewards](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L493)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

This is not an attack, it's low.
1. **Owner** calls `setTVSAllocation` or `setStablecoinAllocation` for a `rewardProjectId` that already has KOL allocations set.
2. The function overwrites `kolTVSRewards[kol]` or `kolStablecoinRewards[kol]` with new values.
3. Previously allocated rewards stored for that KOL are no longer referenced anywhere and cannot be withdrawn.
4. The contract still holds the previously transferred tokens, resulting in stuck funds.


### Impact

The tokens stuck in the contract aren't lost, because the admin can use the `withdrawStuckTokens` function to withdraw them.

### PoC

_No response_

### Mitigation

In order to mitigate the issue, consider one of the following solutions:
1. Clear and refund any previously allocated but unclaimed rewards.
2. Update allocations using a delta-based model rather than overwriting.
3. Revert when allocations already exist unless they are explicitly reset.
  