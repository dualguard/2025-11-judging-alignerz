# [000096] Unauthenticated DoS in `distributeRewardTVS` and `distributeStablecoinAllocation` Due to Array Length Mismatch
  
  ### Summary

The incorrect loop bounds in `distributeRewardTVS` and `distributeStablecoinAllocation` will cause transaction reverts when the owner attempts to distribute unclaimed allocations to specific KOLs, as the functions iterate using the stored array length but index into the calldata parameter array. 

### Root Cause

In `AlignerzVesting` , the `distributeRewardTVS()` and `distributeStablecoinAllocation()` lack access control and use `rewardProject.kolTVSAddresses.length` and `rewardProject.kolStablecoinAddresses.length` as loop bounds, but access the kol calldata array parameter using the loop index i.  When `kol.length < kolTVSAddresses.length`, the function attempts to access `kol[i] where i >= kol.length`, causing an out-of-bounds revert. 

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550

### Internal Pre-conditions

1. Owner needs to call `setTVSAllocation()` or` setStablecoinAllocation()` to allocate tokens to multiple KOLs
2. Some KOLs need to not claim their allocations before the claim deadline
3. Anyone attempts distributing to a subset of unclaimed KOLs (fewer addresses than total unclaimed)

### External Pre-conditions

N/A

### Attack Path

1. Owner sets allocations for 5 KOLs: [kolA, kolB, kolC, kolD, kolE]
2. Only 2 KOLs claim during the claim window (kolA and kolB claim, kolC/kolD/kolE don't)
3. After claim deadline, the internal array contains 3 remaining addresses: kolTVSAddresses = [kolC, kolD, kolE] (length = 3)
4. Anyone calls distributeRewardTVS(0, [kolC]) to distribute only to kolC

- Loop runs with len = 3 (kolTVSAddresses.length) AlignerzVesting.sol:528
- Iteration 0: Calls _claimRewardTVS(0, kol[0]) → works (kolC)
- Iteration 1: Attempts _claimRewardTVS(0, kol[1]) → REVERTS (kol.length = 1, accessing out-of-bounds index)

### Impact

Any can call these functions after the claim deadline, but they will revert unless the `kol` parameter contains exactly as many addresses as there are unclaimed. And, this defeat the purpose of the function to allow selective distribute of `kol` addresses.
Anyone attampt to help in distribute will face reverts. 
The only alleranative is to call `distributeRemainingRewardTVS/distributeRemainingStablecoinAllocation` which ARE owner-only.

### PoC

N/A

### Mitigation

Fix the loop bound 
```diff
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {  
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];  
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());  
-     uint256 len = rewardProject.kolStablecoinAddresses.length;
+   uint256 len = kol.length; 
    for (uint256 i; i < len;) {  
        _claimRewardTVS(rewardProjectId, kol[i]);  
        unchecked {  
            ++i;  
        }  
    }  
}
```
  