# [000095] Stale index mapping entries remain after KOL claims allocation
  
  ### Summary

The failure to delete index mapping entries in `_claimRewardTVS` and `_claimStablecoinAllocation` will cause stale data to persist in storage as KOLs claim their allocations, leaving unnecessary mapping entries that waste gas and violate data invariants.

### Root Cause

In `AlignerzVesting` and `AlignerzVesting`, the `_claimRewardTVS()` and `_claimStablecoinAllocation()` incorrectly update `kolTVSIndexOf[kol]` and `kolStablecoinIndexOf[kol]` to `arrayLength - 1` instead of deleting these mapping entries after a KOL claims.  This leaves stale index data in storage for KOLs who have already claimed and are no longer in the array.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L617

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L636

### Internal Pre-conditions

1. Owner needs to call `setTVSAllocation()` or `setStablecoinAllocation()` to set allocations
2. KOL needs to call `claimRewardTVS()` or `claimStablecoinAllocation()` to claim their allocation

### External Pre-conditions

N/A

### Attack Path

1. Owner sets allocations for multiple KOLs via `setTVSAllocation()`
2. KOL calls `claimRewardTVS()` to claim their allocation
3. The function performs swap-and-pop to remove the KOL from `kolTVSAddresses` array
4. Instead of deleting `kolTVSIndexOf[kol]`, the function sets it to` arrayLength - 1`
5. The KOL's index mapping entry remains in storage pointing to an invalid/stale position

### Impact

- Claimed KOLs retain stale index entries, breaking the invariant that index mappings should reflect only unclaimed addresses.
- Creates unnecessary storage writes and long-term wasted gas for any future operations relying on these mappings.
- Leaves confusing and inconsistent state data for both TVS and stablecoin allocations.

### PoC

N/A

### Mitigation

Delete the index mapping entry instead 

_claimRewardTVS()
```diff
+    delete rewardProject.kolTVSIndexOf[kol];  
```

_claimStablecoinAllocation() 
```diff
+   delete rewardProject.kolStablecoinIndexOf[kol];
```
  