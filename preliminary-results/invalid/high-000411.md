# [000411] Stablecoin Distribution Denial of Service via Blacklisted Recipient
  
  ### Summary

The stablecoin distribution functions `distributeStablecoinAllocation` and `distributeRemainingStablecoinAllocation` transfer ERC20 tokens to KOL addresses in a loop without handling transfer failures. If a stablecoin (such as USDC or USDT) has blacklisted one of the KOL addresses, the `safeTransfer` call will revert, aborting the entire batch and preventing distribution to all remaining legitimate recipients.

### Root Cause

A KOL

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Relevant code from distribution functions:
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L577

1. Project sets up stablecoin allocations for KOLs: `[Alice, Bob, Carol, Malicious, Dave, Eve]`
2. Stablecoin issuer (e.g., Circle for USDC) blacklists `Malicious` address due to sanctions/fraud
3. After claim deadline, anyone calls `distributeStablecoinAllocation(projectId, [...])` or owner calls `distributeRemainingStablecoinAllocation(projectId)`
4. Loop processes Alice ✓, Bob ✓, Carol ✓
5. Loop attempts transfer to `Malicious` → stablecoin contract reverts (address blacklisted)
6. **Entire transaction reverts** → Dave and Eve never receive their allocations
7. No partial progress saved; subsequent retry attempts will fail at the same point

Known blacklist-enabled stablecoins:
- USDC (Circle's blacklist)
- USDT (Tether's blacklist)
- Other regulated stablecoins

### Impact

- **Complete distribution halt**: One blacklisted address blocks all subsequent distributions
- **Funds locked**: Legitimate KOLs after the blacklisted entry cannot receive their allocations
- **Griefing potential**: Malicious KOL can deliberately get blacklisted to DoS the distribution process

### PoC

_No response_

### Mitigation


```diff
 function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
     RewardProject storage rewardProject = rewardProjects[rewardProjectId];
     require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
     uint256 len = rewardProject.kolStablecoinAddresses.length;
     for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
         address kol = rewardProject.kolStablecoinAddresses[i];
         uint256 amount = rewardProject.kolStablecoinRewards[kol];
         rewardProject.kolStablecoinRewards[kol] = 0;
-        rewardProject.stablecoin.safeTransfer(kol, amount);
+        // Use try/catch to handle blacklisted addresses
+        try rewardProject.stablecoin.transfer(kol, amount) returns (bool success) {
+            if (!success) {
+                emit StablecoinAllocationsDistributed(rewardProjectId, kol, 0); // Indicate skip
+            } else {
+                emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
+            }
+        } catch {
+            // Transfer failed (blacklisted or other issue), skip this recipient
+            emit StablecoinAllocationsDistributed(rewardProjectId, kol, 0); // Indicate skip
+        }
         rewardProject.kolStablecoinAddresses.pop();
-        emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
         unchecked { --i; }
     }
 }
```

  