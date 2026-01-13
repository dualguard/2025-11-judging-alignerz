# [000400] The wrong array length is used for iteration in distributeRewardTVS()
  
  ### Summary

The `distributeRewardTVS()` function in the AlignerZ reward distribution contract contains a critical logic bug that can lead to missed distributions, silent reward locking, and unexpected reverts.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525C5-L536C1?plain=1

### Root Cause

The loop runs based on `kolTVSAddresses.length`, not `kol.length`. If `kolTVSAddresses` is shorter than kol, some addresses in kol are skipped. If it's longer, the loop will attempt to access `kol[i]` beyond its bounds, causing a runtime revert due to array index out of range

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked { ++i; }
    }
}
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

- Admin may unintentionally lock rewards permanently.
- Distribution can revert, preventing completion.
- Integrity of reward distribution is compromised.

### PoC

_No response_

### Mitigation

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());

    uint256 len = kol.length;
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked { ++i; }
    }
}
```

Additional to ensure correctness and prevent misuse. Check that input list matches expected list:

```solidity
require(len == rewardProject.kolTVSAddresses.length, Wrong_KOL_List());
```
  