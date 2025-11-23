# [000534] Stablecoin distribution will revert if KOL is blacklisted for USDC/USDT
  
  ## Summary

Stablecoin distribution in the function `distributeRemainingStablecoinAllocation()` and `distributeStablecoinAllocation()` will revert if if at least one of the KOL is blacklisted for USDC/USDT.



## Root cause

The stablecoins USDC and USDT have a blacklist compliance mechanism. If an address is blacklisted, transfers will be reverted, breaking the `distributeRemainingStableCoinAllocation()` and `distributeStablecoinAllocation()` execution.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L589C1-L590C1

```solidity
 function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
            address kol = rewardProject.kolStablecoinAddresses[i];
            uint256 amount = rewardProject.kolStablecoinRewards[kol];
            rewardProject.kolStablecoinRewards[kol] = 0;
@=>            rewardProject.stablecoin.safeTransfer(kol, amount);
            rewardProject.kolStablecoinAddresses.pop();
            emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
            unchecked {
                --i;
            }
        }
    }
```



https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L633

```solidity
    function _claimStablecoinAllocation(uint256 rewardProjectId, address kol) internal {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        uint256 amount = rewardProject.kolStablecoinRewards[kol];
        require(amount > 0, Caller_Has_No_Stablecoin_Allocation());
        rewardProject.kolStablecoinRewards[kol] = 0;
@=>        rewardProject.stablecoin.safeTransfer(kol, amount);
        uint256 index = rewardProject.kolStablecoinIndexOf[kol];
        uint256 arrayLength = rewardProject.kolStablecoinAddresses.length;
        rewardProject.kolStablecoinIndexOf[kol] = arrayLength - 1;
        address lastIndexAddress = rewardProject.kolStablecoinAddresses[arrayLength - 1];
        rewardProject.kolStablecoinIndexOf[lastIndexAddress] = index;
        rewardProject.kolStablecoinAddresses[index] = rewardProject.kolStablecoinAddresses[arrayLength - 1];
        rewardProject.kolStablecoinAddresses.pop();
        emit StablecoinAllocationClaimed(rewardProjectId, kol, amount);
    }
    
    // ......
    
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```





## Impact

This issue prevents receiving funds for KOLs in some cases and leads to the loss of funds. However, the likelihood of this happening is low.



## Mitigation

```solidity
    function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
            address kol = rewardProject.kolStablecoinAddresses[i];
            uint256 amount = rewardProject.kolStablecoinRewards[kol];
            rewardProject.kolStablecoinRewards[kol] = 0;
+            try rewardProject.stablecoin.transfer(kol, amount) {
+            } catch {
+                //skip blacklisted user
+            }
            rewardProject.kolStablecoinAddresses.pop();
            emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
            unchecked {
                --i;
            }
        }
    }
```



```solidity
    function _claimStablecoinAllocation(uint256 rewardProjectId, address kol) internal {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        uint256 amount = rewardProject.kolStablecoinRewards[kol];
        require(amount > 0, Caller_Has_No_Stablecoin_Allocation());
-        rewardProject.kolStablecoinRewards[kol] = 0;
+        try rewardProject.stablecoin.transfer(kol, amount) {
+        } catch {
+            //skip blacklisted user
+            return;
+        }
+		rewardProject.kolStablecoinRewards[kol] = 0;
        uint256 index = rewardProject.kolStablecoinIndexOf[kol];
        uint256 arrayLength = rewardProject.kolStablecoinAddresses.length;
        rewardProject.kolStablecoinIndexOf[kol] = arrayLength - 1;
        address lastIndexAddress = rewardProject.kolStablecoinAddresses[arrayLength - 1];
        rewardProject.kolStablecoinIndexOf[lastIndexAddress] = index;
        rewardProject.kolStablecoinAddresses[index] = rewardProject.kolStablecoinAddresses[arrayLength - 1];
        rewardProject.kolStablecoinAddresses.pop();
        emit StablecoinAllocationClaimed(rewardProjectId, kol, amount);
    }
```

  