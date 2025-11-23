# [000318] Stale KOL index mapping after claim in AlignerzVesting reward module
  
  ### Description:
In `AlignerzVesting`, the KOL index mappings `rewardProject.kolTVSIndexOf` and `rewardProject.kolStablecoinIndexOf` are used to support swap‑and‑pop removal from the `kolTVSAddresses` / `kolStablecoinAddresses` arrays.

When a KOL claims TVS rewards in [`_claimRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L601-L623) :

```solidity
uint256 index = rewardProject.kolTVSIndexOf[kol];
uint256 arrayLength = rewardProject.kolTVSAddresses.length;
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
rewardProject.kolTVSAddresses.pop();
```

and similarly in [`_claimStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L628-L642) for `kolStablecoinAddresses`.

After the `pop()`, the array length becomes `arrayLength - 1`, but `rewardProject.kolTVSIndexOf[kol]` remains set to `arrayLength - 1`, which is now out of bounds and never used again. This leaves a **stale, incorrect index** entry for each KOL that has already claimed, which can be misleading for off‑chain tooling or for future contract versions that might rely on the mapping’s correctness.

### Impact:
Leaves stale `kolTVSIndexOf` / `kolStablecoinIndexOf` entries for claimed KOLs, which can cause confusion and makes these mappings unsafe to reuse in upgrades or extensions.

### Recommendation:
After a KOL has claimed and been removed from the corresponding address array, clear their index mapping entry instead of assigning a bogus index, for example:

```solidity
// After performing the swap-and-pop on kolTVSAddresses:
delete rewardProject.kolTVSIndexOf[kol];

// And similarly in _claimStablecoinAllocation():
delete rewardProject.kolStablecoinIndexOf[kol];
```

This keeps the mappings consistent with their documented meaning and avoids stale state for claimed KOLs.



  