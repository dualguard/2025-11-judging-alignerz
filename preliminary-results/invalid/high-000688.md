# [000688] Missing `onlyOwner` modifier in manual distribution functions allows unauthorized actors to trigger distributions
  
  ### Summary

The functions `distributeRewardTVS` and `distributeStablecoinAllocation` are intended to be administrative functions callable only by the contract owner, as indicated by their comment and the behavior of their counterpart "sweep" functions (`distributeRemainingRewardTVS` / `distributeRemainingStablecoinAllocation`). However, they lack the `onlyOwner` modifier. This oversight allows any external user to trigger the distribution logic, potentially disrupting the project's operational schedule.

### Root Cause

In `AlignerzVesting.sol`, the functions `distributeRewardTVS` and `distributeStablecoinAllocation` are declared as `external` but missing the access control modifier `onlyOwner`.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L522-L551
This contradicts the function documentation:
```solidity
/// @notice Allows the owner to distribute ...
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external { // Missing onlyOwner
```

And stands in contrast to the logic of the "remaining" distribution functions, which correctly enforce access control:
```solidity
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner { ... }
```

### Internal Pre-conditions

1.  The `claimDeadline` for the specific `rewardProjectId` must have passed.
2.  There must be KOLs who have not yet claimed their allocations.

### External Pre-conditions

_No response_

### Attack Path

1.  The `claimDeadline` passes.
2.  The Owner intends to distribute rewards to a specific batch of users at a specific time .
3.  An attacker (or any random user) calls `distributeRewardTVS` or `distributeStablecoinAllocation`.
4.  The transaction executes successfully, forcing the distribution to happen immediately, bypassing the Owner's control.

### Impact

The protocol owner loses control over the timing and execution of reward distributions. 

### PoC

_No response_

### Mitigation

Add the `onlyOwner` modifier to both `distributeRewardTVS` and `distributeStablecoinAllocation` to ensure they match the intended design.
  