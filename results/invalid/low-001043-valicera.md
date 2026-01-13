# [001043] _claimRewardTVS and _claimStablecoinAllocation functions leave a stale index inside map
  
  ### Summary

Both functions `_claimRewardTVS` and `_claimStablecoinAllocation` swap-pop's the kolTVSAddresses array but never clears (delete) the mapping entry kolTVSIndexOf[kol] for the removed address, leaving a stale index value pointing outside the shrunken array.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L601-L642

### Root Cause

The removal code implements a swap‑and‑pop but mishandles the mapping for the removed address. Instead of deleting the mapping entry for the removed address it writes/updates indices for the moved element and leaves the removed address' mapping set to a now-out-of-range value.
Concretely it sets kolTVSIndexOf[kol] = arrayLength - 1 (or never deletes kolTVSIndexOf[removedAddress]) and then pops the array.
```solidity
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
```
```solidity
rewardProject.kolStablecoinIndexOf[kol] = arrayLength - 1;
```
The mapping for the removed address remains pointing to the old index.

### Internal Pre-conditions

1. Rewards of TVS or Stablecoin allocations must be claimed.

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Stale mapping entries can return out-of-range indices later, causing incorrect swap/pop behavior, corrupted address -> index invariants, unexpected state (wrong address moved), and potential reverts or logic errors when the stale index is used by other functions.

### PoC

Consider the following scenario:
Start:
 addresses = [A, B, C], mappings A -> 0, B -> 1, C -> 2, arrayLength = 3.
Call _claimRewardTVS with kol == C (removing last element):
        index = kolTVSIndexOf[C] = 2
        arrayLength = 3
        code sets kolTVSIndexOf[C] = arrayLength - 1 = 2 (no change)
        lastIndexAddress = kolTVSAddresses[2] = C
        code sets kolTVSIndexOf[lastIndexAddress] = index (still C -> 2)
        writes kolTVSAddresses[index] = kolTVSAddresses[arrayLength - 1] (no-op)
        pop() removes C from the array -> addresses = [A, B] (length 2)
Result:
array no longer contains C, but kolTVSIndexOf[C] == 2 remains -> out of range
(valid indices are 0..1). pop removed the array element; the mapping was never deleted.

### Mitigation

Use a correct swap-and-pop plus delete pattern:

When removing, swap the last element into the removed slot and update the moved address mapping to the new index.
After pop(), explicitly delete rewardProject.kolTVSIndexOf[removedAddress] to remove the stale mapping entry.
  