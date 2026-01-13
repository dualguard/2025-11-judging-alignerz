# [000999] Missing Access Control Allows Anyone to Distribute Remaining Rewards
  
  ### Summary

A lack of access control on the functions distributeRewardTVS() and distributeStablecoinAllocation() will cause unauthorized distribution of reward tokens for all KOLs, as anyone can call these functions after the claim window and trigger token transfers through the internal _claimRewardTVS() and _claimStablecoinAllocation() functions.

### Root Cause

In RewardDistributor.sol (lines where the functions are defined), the developer intended these functions to be owner-only, as stated in the NatSpec comments:
```solidity 
/// @notice Allows the owner to distribute ...
```

However, both functions are declared as:
```solidity
function distributeRewardTVS(...) external { ... }
function distributeStablecoinAllocation(...) external { ... }
```

with no onlyOwner modifier, making them callable by any address.

Thus, the root cause is:
A missing access control check (onlyOwner) on two functions that should be restricted to the owner/admin.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

*

### Impact

The protocol and owner lose control over the distribution of all unclaimed KOL rewards.

### PoC

_No response_

### Mitigation

add ownership restrictions 
```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol)
    external
    onlyOwner
{
    ...
}

function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol)
    external
    onlyOwner
{
    ...
}
```
  