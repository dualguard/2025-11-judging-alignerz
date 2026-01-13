# [000785] Storage Array Pollution via Duplicate Allocation Calls
  
  ### Summary

Lack of duplicate prevention in allocation functions will cause reward distribution to fail for the protocol team as the owner can call `setTVSAllocation` and `setStablecoinAllocation` multiple times, pushing duplicate KOL addresses to arrays and breaking downstream distribution logic.

### Root Cause

In `AlignerzVesting.sol` the functions `setTVSAllocation` and `setStablecoinAllocation` unconditionally push KOL addresses to the `kolTVSAddresses` and `kolStablecoinAddresses` arrays respectively without checking if the address already exists. Since these functions can be called multiple times by the owner, duplicate entries accumulate in the arrays, causing downstream functions that iterate over these arrays to either revert or execute incorrectly.

[pushing of duplicate entries in array](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L466)


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls `setTVSAllocation` with a KOL address, pushing it to `kolTVSAddresses` and setting `kolTVSRewards[kol]` to the allocation amount
2. Owner calls `setTVSAllocation` again with the same KOL address (either intentionally or by mistake), pushing a duplicate entry to `kolTVSAddresses` and overwriting `kolTVSRewards[kol]` with a new amount
3. When `distributeRewardTVS` is called, it iterates over `kolTVSAddresses.length`, which now exceeds the number of unique KOLs
4. On the second iteration for the duplicate address, `_claimRewardTVS` is called again but `kolTVSRewards[kol]` has already been zeroed out in the first iteration
5. The require statement `require(amount > 0, Caller_Has_No_TVS_Allocation())` fails, causing the entire distribution to revert
6. Alternatively, if `distributeRemainingRewardTVS` is called, it will mint multiple NFTs for the same KOL due to the duplicates in the array, resulting in the KOL receiving more allocations than intended


### Impact

The protocol cannot successfully distribute rewards to KOLs. The owner suffers a functional loss as reward distribution transactions revert when duplicate allocations exist. Additionally, if alternative distribution functions are used, duplicate KOL entries cause unintended multiple NFT minting and over-allocation of rewards to duplicated addresses, resulting in protocol loss of up to the amount of re-allocated rewards to the duplicated entries.


### PoC

_No response_

### Mitigation

Add a check in both `setTVSAllocation` and `setStablecoinAllocation` to prevent duplicate KOL addresses from being added to the arrays. Before pushing a KOL address, verify that the address does not already exist in the array (e.g., by checking if `kolTVSIndexOf[kol]` is uninitialized or by maintaining a mapping to track which addresses have been allocated). Alternatively, clear and reinitialize the arrays on each call to these functions, or add a separate function to remove duplicate entries before distribution occurs.
  