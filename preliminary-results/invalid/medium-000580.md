# [000580] Incorrect Index Mapping in Batch Allocation Functions Causes State Corruption and Permanent Claim Mechanism Failure
  
  
## Summary

Using the loop counter instead of the actual array index when setting KOL index mappings in `setTVSAllocation()` and `setStablecoinAllocation()` causes state corruption and permanent claim failures for KOLs when these functions are called multiple times, as the swap-and-pop logic in `_claimRewardTVS()` and `_claimStablecoinAllocation()` removes the wrong KOL from the array.

## Root Cause

In `AlignerzVesting.sol:469` and `AlignerzVesting.sol:494`, the index mapping uses the loop counter `i` instead of the actual array position after pushing to the array.

- In `setTVSAllocation()` at line 469: `rewardProject.kolTVSIndexOf[kol] = i;` uses the loop counter `i` (0, 1, 2...) instead of the array index after `push()`.
- In `setStablecoinAllocation()` at line 494: `rewardProject.kolStablecoinIndexOf[kol] = i;` has the same issue.

When called multiple times, the loop counter resets to 0, but the array continues growing. New KOLs are pushed at indices 3, 4, 5..., while the mapping stores 0, 1, 2..., causing a mismatch. This can happen when the list of KOLs is too long for a single `setTVSAllocation()` or `setStablecoinAllocation()` call without reverting.

## Internal Pre-conditions

1. Owner calls `setTVSAllocation()` or `setStablecoinAllocation()` at least once to add initial KOLs.
2. Owner calls the same function again to add more KOLs (e.g., to avoid out-of-gas errors with large batches).
3. At least one KOL from the second batch attempts to claim their allocation.

## External Pre-conditions

None required.

## Attack Path

1. Owner calls `setTVSAllocation(rewardProjectId, totalTVSAllocation1, vestingPeriod, [kol1, kol2, kol3], [amount1, amount2, amount3])`.
   - Array indices: kol1=0, kol2=1, kol3=2
   - Mappings: `kolTVSIndexOf[kol1]=0`, `kolTVSIndexOf[kol2]=1`, `kolTVSIndexOf[kol3]=2` ✓
2. Owner calls `setTVSAllocation(rewardProjectId, totalTVSAllocation2, vestingPeriod, [kol4, kol5], [amount4, amount5])`.
   - Array indices after push: kol4=3, kol5=4
   - Mappings: `kolTVSIndexOf[kol4]=0` ❌, `kolTVSIndexOf[kol5]=1` ❌
3. kol4 calls `claimRewardTVS(rewardProjectId)`.
   - Line 615: `index = kolTVSIndexOf[kol4]` returns 0 (wrong)
   - Line 620: `kolTVSAddresses[0] = kolTVSAddresses[arrayLength-1]` overwrites kol1 with the last address
   - Line 619: `kolTVSIndexOf[lastIndexAddress] = 0` sets the wrong index for the swapped address
   - State is corrupted: kol1's index mapping is wrong, and kol4 is not removed correctly
4. Subsequent claims by kol1, kol2, kol3, or kol5 fail or remove the wrong KOLs, permanently breaking the claim mechanism and corrupting the contract state.

## Impact

1. The swap-and-pop logic removes the wrong addresses, corrupting index mappings and causing:
- Incorrect KOLs being removed from the array
- Permanent claim failures for multiple KOLs

2. Affected KOLs cannot claim their TVS or stablecoin allocations if the `kolTVSIndexOf` mapping for the KOL or `lastIndexAddress` returns an index that results in an out-of-bound revert when accessing `kolTVSIndexOf` .

The protocol suffers a loss of functionality, and affected KOLs lose access to their allocations.

## Proof of Concept
N/A

## Mitigation

Use the actual array index after pushing instead of the loop counter:

```js
// In setTVSAllocation() at line 469:
rewardProject.kolTVSIndexOf[kol] = rewardProject.kolTVSAddresses.length - 1;

// In setStablecoinAllocation() at line 494:
rewardProject.kolStablecoinIndexOf[kol] = rewardProject.kolStablecoinAddresses.length - 1;
```
  