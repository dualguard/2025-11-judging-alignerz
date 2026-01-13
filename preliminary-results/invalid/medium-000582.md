# [000582] Incorrect Index Mapping Update After Array Element Removal Causes Stale State and Unnecessary Gas Consumption in `_claimRewardTVS()`
  
  
## Summary

Updating the KOL's index mapping to the last array position before removing them from the array in `_claimRewardTVS()` causes stale state and wastes gas for the protocol, as the function sets an invalid index for an address no longer in the array.

## Root Cause

In [`AlignerzVesting.sol:617`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L617), the code sets `rewardProject.kolTVSIndexOf[kol] = arrayLength - 1` before removing the KOL from the array via `pop()` at line 621. After removal, the array length decreases, so the stored index points to an invalid position. The KOL is no longer in the array, so their index mapping should not reference any array position.

## Internal Pre-conditions

1. A KOL needs to have a TVS allocation set via `setTVSAllocation()` so they exist in `rewardProject.kolTVSAddresses`
2. The KOL needs to call `claimRewardTVS()` or the owner needs to call `distributeRewardTVS()` to trigger `_claimRewardTVS()`
3. The KOL must not have already claimed their reward (their `kolTVSRewards[kol]` must be greater than 0)

## External Pre-conditions

None

## Attack Path

1. Owner calls `setTVSAllocation()` to allocate TVS rewards to multiple KOLs, populating `rewardProject.kolTVSAddresses` and `kolTVSIndexOf` mappings
2. A KOL calls `claimRewardTVS()` (or owner calls `distributeRewardTVS()`), which invokes `_claimRewardTVS()`
3. The function retrieves the KOL's current index from `kolTVSIndexOf[kol]` at line 615
4. The function sets `kolTVSIndexOf[kol] = arrayLength - 1` at line 617, updating the mapping to point to the last position
5. The function performs swap-and-pop: moves the last element to the KOL's position and removes the last element via `pop()` at line 621
6. After removal, the array length is reduced by 1, but `kolTVSIndexOf[kol]` still contains `arrayLength - 1` (now an invalid index)
7. The stale index mapping remains in storage, consuming gas and representing incorrect state

## Impact

- The protocol maintains incorrect state: `kolTVSIndexOf[kol]` points to an invalid array position after the KOL is removed
- Unnecessary gas consumption: a storage write (SSTORE) updates a mapping that will never be used again since the KOL is no longer in the array
- Potential confusion: if the index mapping is read by external systems, it would return an invalid value

## Proof of Concept
N/A

## Mitigation

Remove the unnecessary index update for the KOL being removed, since they are no longer in the array:

```diff
function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
    // ...
-    rewardProject.kolTVSIndexOf[kol] = arrayLength - 1; // Remove line 617
    // ...
}
```

**Note:** The same issue exists in `_claimStablecoinAllocation()` at line 636 and should be fixed using the same approach.
  