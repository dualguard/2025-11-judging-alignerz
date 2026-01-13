# [000970] Missing Access Control in `distributeRewardTVS()` and `distributeStablecoinAllocation()`
  
  ### Summary

The functions `distributeRewardTVS()` and `distributeStablecoinAllocation()` in `AlignerzVesting` are documented as owner-only operations in their NatSpec comments, but lack the `onlyOwner` modifier. Any address can call these functions after the claim deadline,  disrupting the intended reward distribution flow.

### Root Cause

```solidity 
>>> /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
function distributeRewardTVS(...)
>>> external {
  ///...code }
```

```solidity 
>>> /// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
function distributeStablecoinAllocation(...)
>>> external {
    ///...code
}
```

Missing access control: Both functions lack onlyOwner modifier despite NatSpec stating "Allows the owner to distribute..."

Inconsistency: Similar function `distributeRemainingRewardTVS()` correctly uses onlyOwner modifier


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Wait for the requirement period to end
2. Call these functions 

### Impact

- Any address can trigger reward distribution after deadline
- Owner loses control over timing and order of distribution

### PoC

_No response_

### Mitigation

Add the `onlyOwner` modifier to these functions or change the NatSpec comment so that it does not mislead auditors and developers (if it is a design decision to allow anyone to call these functions). 

```solidity 
/// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
function distributeStablecoinAllocation(...)
>>> external onlyOwner {
    ///...code
}
```
  