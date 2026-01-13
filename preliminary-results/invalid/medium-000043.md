# [000043] `distributeRewardTVS` and `distributeStablecoinAllocation` can be called by anyone
  
  ### Summary

The functions `distributeRewardTVS` and `distributeStablecoinAllocation` are publicly callable, meaning anyone can invoke them. However, the comments above both functions state that they should be admin-only and invoked by the owner:
```
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
```

and:

```
/// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
```

This mismatch suggests the functions should include an access control modifier (e.g., onlyOwner) but currently do not.

### Root Cause

No `onlyOwner` modifier on `distributeRewardTVS` and `distributeStablecoinAllocation`
* https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L525
* https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L540

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

The owner's functions (`distributeRewardTVS` and `distributeStablecoinAllocation`) are callable by anyone

### PoC

_No response_

### Mitigation

Consider adding an `onlyOwner` modifier.
  