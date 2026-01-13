# [001001] `distributeRewardTVS` and `distributeStablecoinAllocation` use incorrect loop bound, causing reverts or incomplete distribution.
  
  ### Summary

The functions use `rewardProject.kolTVSAddresses.length` (the number of unclaimed users) as the loop bound for iterating over the input array `kol` (the users to distribute to). This mismatch will cause an index out of bounds revert if the storage array is larger than the input array, or fail to process all input addresses if the storage array is smaller, can be happen also if owner blacklsit some users.

### Root Cause

In `AlignerzVesting.sol:673` and `AlignerzVesting.sol:688`:
```solidity
uint256 len = rewardProject.kolTVSAddresses.length; // Wrong length source
for (uint256 i; i < len;) {
    _claimRewardTVS(rewardProjectId, kol[i]); // Accessing input array
    // ...
}
```
while using the full of the tvsAddresses it can also run out of gas and revert 
It should use `kol.length`

### Internal Pre-conditions

 The length of the input array `kol` is different from the number of unclaimed users in storage.

### External Pre-conditions

None.

### Attack Path

1. Owner calls `distributeRewardTVS` with a single address `[Alice]`.
2. There are 10 unclaimed users in storage. `len` = 10.
3. Loop runs for `i=0`. `kol[0]` (Alice) processed.
4. Loop runs for `i=1`. `kol[1]` -> Revert (Index out of bounds).

### Impact

The function is unreliable and likely to revert, preventing the owner from distributing tokens to specific subsets of users.

### PoC

_No response_

### Mitigation

Change `uint256 len = rewardProject.kolTVSAddresses.length;` to `uint256 len = kol.length;`.
  