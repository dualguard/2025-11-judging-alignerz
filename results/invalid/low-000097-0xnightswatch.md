# [000097] Owner will mint duplicate empty NFTs when calling setTVSAllocation() multiple times.
  
  ### Summary

The lack of array clearing in `setTVSAllocation` will cause duplicate empty NFT minting for KOLs as the owner will call `setTVSAllocation()` multiple times to update allocations, then call `distributeRemainingRewardTVS()` which mints NFTs for all array entries including duplicates.

### Root Cause

In `AlignerzVesting.sol`  the setTVSAllocation function appends addresses to the `kolTVSAddresses` array without clearing previous entries. When the function is called multiple times for the same rewardProjectId to update allocations, duplicate addresses accumulate in the array while the `kolTVSRewards` mapping gets overwritten with the latest values. 

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L466


### Internal Pre-conditions

1. Owner needs to call `setTVSAllocation()` for reward project to set intial allocations
2. Owner need to call `setTVSAllocation()` again for the same `rewardProjectID` to update allocation.
3. Owner nees to can `distributeRemainingRewardsTVS()` after the claim deadline for users have  not claimed. (If the user has duplicate addresses, user wont be able to claim individually becuase there a amount > 0 in `_claimRewardTVS()` that is not present in `distributeRemainingRewardTVS()`.

### External Pre-conditions

N/A

### Attack Path

1. Initial allocation:
   - setTVSAllocation(0, 1000e18, 365 days, [A, B], [500e18, 500e18])
   - kolTVSAddresses = [A, B]
   - kolTVSRewards[A] = 500e18
   - kolTVSRewards[B] = 500e18

2. Updated allocation (values overwritten, addresses appended):
   - setTVSAllocation(0, 1200e18, 365 days, [A, B], [600e18, 600e18])
   - kolTVSAddresses = [A, B, A, B]   // duplicates accumulate
   - kolTVSRewards[A] = 600e18        // previous values overwritten
   - kolTVSRewards[B] = 600e18

3. distributeRemainingRewardTVS(0) after deadline:
   - Iteration 1: A → mints NFT(600e18), kolTVSRewards[A] = 0
   - Iteration 2: B → mints NFT(600e18), kolTVSRewards[B] = 0
   - Iteration 3: A → mints NFT(0) due to previous zeroing
   - Iteration 4: B → mints NFT(0) due to previous zeroing

Note that in 3rd step, address A abd B can claim the first two iterations (1&2) individually but not the last two iterations (3 & 4) because `_claimRewards()` reverts if the amount is zero.


### Impact

Each affected KOL will recieve duplicate NFTs - one valid with their allocation and one or more with 0 token allocated. 
No funds at lost but this may create confusin, pollutes the NFT collection with worthless tokens, and waste unnecessary gas. 

Number of worthless NFT = numer of time `setTVSAllocation()` - 1 

### PoC

_No response_

### Mitigation

Two options:
1. Clear the array `rewardProject.kolTVSAddresses`  before setting a new allocation. 
``` diff
       function setTVSAllocation( ...) external onlyOwner {  
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];  
      
    // Clear existing array  
+   delete rewardProject.kolTVSAddresses;  
      
    rewardProject.vestingPeriod = vestingPeriod;  
    uint256 length = kolTVS.length;  
    require(length == TVSamounts.length, Array_Lengths_Must_Match());  
      

````

2. Add check to `distributeRemainingRewardTVS()` to skip zero allocations. 

```diff
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner {  
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];  
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());  
    uint256 len = rewardProject.kolTVSAddresses.length;  
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {  
        address kol = rewardProject.kolTVSAddresses[i];  
        uint256 amount = rewardProject.kolTVSRewards[kol];  
          
+        // Skip if amount is 0 (duplicate entry)  
+        if (amount == 0) {  
+           rewardProject.kolTVSAddresses.pop();  
+           unchecked { --i; }  
+           continue;  
+       }  
          
        rewardProject.kolTVSRewards[kol] = 0;  
        // ... rest of minting logic  
    }  
}
```
  