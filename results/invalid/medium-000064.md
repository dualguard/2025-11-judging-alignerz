# [000064] Missing Access Control in `distributeStablecoinAllocation()` Allows Anyone to Force Distribution
  
  ## Summary

The [`distributeStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540) function lacks any access control, allowing anyone to call it and force distribution of stablecoins to KOLs after the claim deadline. According to the contest documentation and walkthrough video, this function should be restricted to the project owner, but the implementation is missing the `onlyOwner` modifier.

## Vulnerability Details

### Root Cause

The function is declared as `external` without any access control modifier:

```solidity
/// @notice Allows the owner to distribute the stablecoin that have not been claimed yet to the KOLs
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
    // ...
}
```

## Impact

Any address can force distribution of stablecoins to KOLs after the claim deadline, removing the owner's control over when distributions occur.
The similar function `distributeRemainingStablecoinAllocation()` has proper `onlyOwner` protection, creating an inconsistent security model where token distributions are protected but stablecoin distributions are not.

## Recommended Mitigation

Add the `onlyOwner` modifier to restrict access to the contract owner

This matches the access control pattern used in `distributeRemainingRewardTVS()` and aligns with the documented intent that only the owner should control distributions.

  