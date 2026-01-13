# [000232] State Inconsistency in Reward Claim Functions due to Flawed Logic
  
  ### Summary

The functions that handle reward claims (`_claimRewardTVS` and `_claimStablecoinAllocation`) contain a logic bug. When a user claims their reward, the function attempts to remove them from a list of eligible claimants. However, the code does this incorrectly and fails to properly clean up its internal records. This leaves incorrect, "stale" data in the contract's state. This corrupts the contracts state.

### Root Cause

in `AlignerzVesting.sol: 617 - 621, 636 - 640`

The problem is in the implementation of the "swap-and-pop" pattern, a common technique to remove an item from an array. The code is supposed to:

1. Move the last person in the list to the position of the person who just claimed.
2. Update the records for the person who was moved.
3. Delete the records for the person who claimed.

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L617-L621

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L636-L640

The code fails on step 3. Instead of deleting the claimant's record from the `kolTVSIndexOf` mapping, the line `rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;` incorrectly updates it with a wrong value, leaving stale data behind.

### Internal Pre-conditions

- A `RewardProject` has been set up.
- A user (let's call them a "KOL") has a valid, unclaimed reward.
- The KOL's address is present in the `kolTVSAddresses` or `kolStablecoinAddresses` array.

### External Pre-conditions

The KOL calls either `claimRewardTVS` or `claimStablecoinAllocation` to get their reward.

### Attack Path

This is a bug triggered by normal user behavior, not a malicious attack.

1. __Initial State__:

- The list of claimants is: `[Alice, Bob, Charlie]`.
- The contract's records are: `{ Alice: 0, Bob: 1, Charlie: 2 }`.
2. __Action__:

- Bob (the user in the middle) calls the function to claim his reward.
3. __Execution Flow__:

- The function processes Bob's claim.
- It moves `Charlie` (the last user) to Bob's old spot. The list becomes `[Alice, Charlie]`.
- It incorrectly updates Bob's record to point to index `2`.
- It correctly updates Charlie's record to point to index `1`.
4. __Final Corrupted State__:

- The list of claimants is now `[Alice, Charlie]`. (Correct)
- The contract's records are now `{ Alice: 0, Bob: 2, Charlie: 1 }`. (Incorrect!)

The record for `Bob` should have been deleted, but it now contains stale data pointing to an index that is out of bounds.

### Impact

The impact is State Corruption. The contract's internal data becomes unreliable and inconsistent. While this doesn't cause a direct loss of funds today, it is a ticking time bomb, future contract upgrades especially rely on state data consistency and an anomaly like this could cause unexpected behaviours.

### PoC

_No response_

### Mitigation

The logic must be fixed to correctly implement the swap-and-pop pattern. This involves deleting the record of the user who claimed and properly handling the case where the claimant is the last person on the list.
  