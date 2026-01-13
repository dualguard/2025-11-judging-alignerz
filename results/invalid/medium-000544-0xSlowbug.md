# [000544] Missing Access Control: Anyone can prematurely distribute stablecoin allocations for KOLs after the claim deadline
  
  ### Summary

A **missing access-control check** in [distributeStablecoinAllocation()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550) will cause **unauthorized distribution** of KOL stablecoin rewards for **all KOLs** as **any external caller** will trigger `_claimStablecoinAllocation()` on their behalf **after the claim deadline**.


### Root Cause

In `rewardDistributor.sol:distributeStablecoinAllocation()` the function lacks an `onlyOwner` (or equivalent) modifier even though the NatSpec states it "Allows the owner to distribute…".
Because of the missing access control, **any address** can call it:

```solidity
function distributeStablecoinAllocation(...) external { ... }
```

### Internal Pre-conditions

1. The claim deadline for the given `rewardProjectId` must have passed (`block.timestamp > claimDeadline`).

### External Pre-conditions

None

### Attack Path

1. Claim deadline passes for a given `rewardProjectId`.
2. Attacker calls:

   ```solidity
   distributeStablecoinAllocation(rewardProjectId, kolArray);
   ```
3. The contract loops and calls `_claimStablecoinAllocation()` for each KOL.
4. The attacker effectively forces claim execution and token transfers without authorization.

### Impact

The Stable coin allocation is meant to be claimed within a particular timeframe as seen in the [claimStablecoinAllocation](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L515-L520). Which only allows claiming before the project claim deadline

```solidity
    function claimStablecoinAllocation(uint256 rewardProjectId) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        @>> require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());
        address kol = msg.sender;
        _claimStablecoinAllocation(rewardProjectId, kol);
    }
```

But by calling `distributeStablecoinAllocation` it bypasses the timeframe requirement and users can claim allocation even after the deadline, making the timeframe restriction ineffective.  

### PoC

_No response_

### Mitigation

Apply proper access control:

```solidity
function distributeStablecoinAllocation(...) 
    external 
    onlyOwner 
{
    ...
}
```
  