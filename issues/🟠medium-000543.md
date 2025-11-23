# [000543] Missing Access Control: Anyone can prematurely distribute TVS rewards for KOLs after the claim deadline
  
  ### Summary

A **missing access-control check** in [distributeRewardTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535) will cause **unauthorized distribution** of unclaimed TVS rewards for **all KOLs**, as **any external address** can trigger `_claimRewardTVS()` on their behalf once the claim window has ended.

### Root Cause


In `rewardDistributor.sol:distributeRewardTVS()` there is **no owner-only restriction**, despite NatSpec and intended semantics indicating it should only be callable by the owner or admin role. The function is declared:

```solidity
function distributeRewardTVS(...) external { ... }
```

Because there is **no `onlyOwner` or equivalent check**, anyone can execute the distribution loop.

---


### Internal Pre-conditions

The claim deadline for the given `rewardProjectId` must have passed (`block.timestamp > claimDeadline`).

### External Pre-conditions

_No response_

### Attack Path

1. Claim deadline passes for a given reward project.
2. Attacker calls:

   ```solidity
   distributeRewardTVS(rewardProjectId, kolArray);
   ```
3. The function loops over all KOL addresses and calls `_claimRewardTVS()` for each entry.
4. TVS rewards are distributed without owner approval or coordination.


### Impact

The TVS reward is meant to be claimed within a particular timeframe as seen in the [claimRewardTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L506-L511). Which only allows claiming before the project claim deadline

But by calling `distributeRewardTVS` it bypasses the timeframe requirement and users can claim allocation even after the deadline, making the timeframe restriction ineffective.  

### PoC

_No response_

### Mitigation

Apply proper access control to restrict the function to the intended owner or operator:

```solidity
function distributeRewardTVS(...) 
    external 
    onlyOwner 
{
    ...
}
```
  