# [000018] Low: Incorrect KOL array handling in `distributeRewardTVS` can cause panic reverts or partial distribution
  
  ### Summary

`distributeRewardTVS` uses `rewardProject.kolTVSAddresses.length` as the loop bound while indexing the caller supplied kol array. The function never enforces that `kol.length` matches `kolTVSAddresses.length`. This mismatch can cause:
- A raw array-out-of-bounds panic if `kol.length < kolTVSAddresses.length`
- Silent partial distribution if `kol.length > kolTVSAddresses.length` (some KOLs are never processed)

### Root Cause

The root cause is inconsistent coupling between storage and external [calldata](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L528-L530):

```solidity
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i; i < len;) { 
        _claimRewardTVS(rewardProjectId, kol[i]);   // <@ indexes into kol
```
Note that: 
- Loop bound is taken from storage (`kolTVSAddresses.length`)
- Indexing is done into calldata (`kol[i]`)
- No invariant enforces `kol.length == kolTVSAddresses.length` or that both lists contain the same addresses

### Internal Pre-conditions

1. A reward project is launched and initializwd
2. `block.timestamp > rewardProject.claimDeadline`, so the `Deadline_Has_Not_Passed()` check passes and `distributeRewardTVS` can be called.
3. `_claimRewardTVS` is the function used to clear rewards and update kolTVSAddresses/kolTVSIndexOf, and it expects its kol argument to correspond to an address that was initialized in kolTVSAddresses / kolTVSIndexOf

### External Pre-conditions

_No response_

### Attack Path

1. Someone calls `vesting.distributeRewardTVS(rewardProjectId, kol);`
2. Inside `distributeRewardTVS`:
- `rewardProject` is loaded from `rewardProjects[rewardProjectId]`
- `require(block.timestamp > rewardProject.claimDeadline, ...)` passes
- `len` is set to `rewardProject.kolTVSAddresses.length`
3. Loop starts:
```solidity
for (uint256 i; i < len;) {
    _claimRewardTVS(rewardProjectId, kol[i]);
    unchecked { ++i; }
}
```
Case A:  `kol.length < len`:
- For `i == 0 ... kol.length - 1`, `kol[i]` is valid
- When `i == kol.length`, `kol[i]` reads past the end of the calldata array -> Solidity generates a `Panic(0x32)` index out-of-bounds revert
- The transaction reverts with a raw panic 

Case B: `kol.length > len`:
- Loop runs only while `i < len` (based on `kolTVSAddresses.length`).
- `kol[len], kol[len+1]`, … are never read
- Only the first `len` entries of `kol` are processed via `_claimRewardTVS`
- Any extra addresses in kol are silently ignored.


### Impact

Impact: low
No loss of funds, however, this leads to incorrect behaviour that is easily corrected 

Likelihood: High
Anyone can call the affected function but requires avoidable bad input 


### PoC

_No response_

### Mitigation

Enforce address equality
```solidity
require(kol[i] == rewardProject.kolTVSAddresses[i], Invalid_KOL_Array_Content());
```
  