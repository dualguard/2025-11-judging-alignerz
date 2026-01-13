# [000889] Any claimer will evict the wrong KOL and strand their allocation
  
  ### Summary

The swap/pop removal without index validation in reward claimers will cause wrong-address eviction and stranded allocations for KOLs as any claimer will pop the address stored at a stale index and leave the rightful KOL in the pending list, corrupting distribution.

### Root Cause

"In `protocol/src/contracts/vesting/AlignerzVesting.sol:615-621` and `:634-640` the swap/pop removal in `_claimRewardTVS` and `_claimStablecoinAllocation` uses `kolTVSIndexOf/kolStablecoinIndexOf` without asserting the index matches the array entry and without deleting the removed index, so if the mapping and array ever desync the code can evict the wrong address and leave stale indices.

### Internal Pre-conditions

1. Storage becomes desynced so `kolTVSIndexOf/kolStablecoinIndexOf` points to a slot holding a different address than the mapping key (e.g., via upgrade/migration script/manual write/past bad swap).
2. At least one pending KOL allocation remains in `kolTVSAddresses` or `kolStablecoinAddresses`.
3. A claim or bulk distribution path is executed for a KOL while the desync exists.

### External Pre-conditions

none

### Attack Path

1. State is already corrupted so `kol*IndexOf[claimer]` points to another address’s slot in the array.
2. The claimer (or owner during bulk distribution) triggers `_claimRewardTVS` or `_claimStablecoinAllocation`.
3. Swap/pop uses the stale index, evicts the other address from the array, and leaves the claimer (and a stale mapping entry) in place.
4. The evicted KOL may later be skipped in bulk distribution or face further mis-removals as corruption cascades.

### Impact

Legitimate KOLs can be silently evicted from pending arrays and skipped in owner distributions, stranding their allocations until manual repair, while the data corruption can propagate to later claims and off-chain indexers; the claimer does not lose funds, but affected KOLs may miss payouts or face reverts when distributions assume intact arrays.

### PoC

_No response_

### Mitigation

Validate the index before swapping (`require(kol*Addresses[index] == kol)`), swap last into index, update that address’s index, pop, and delete the removed mapping entry. Consider adding a view or maintenance script to assert the invariant across reward projects.
  