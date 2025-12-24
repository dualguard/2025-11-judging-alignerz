# [000063] Unauthorized Public Access to distributeRewardTVS Enables Arbitrary Reward Distribution
  
  ## Summary

The [`distributeRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) function is publicly accessible without any access restrictions, enabling any external caller to trigger token distributions to KOLs once the claim window closes. The function's NatSpec documentation and contest materials clearly indicate this should be an owner-only operation, yet the `onlyOwner` modifier is absent.

## Vulnerability Details

### Root Cause

The function is declared as `external` without any access control modifier:

```solidity
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs during the claimWindow
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {}
```

1. Function documentation explicitly declares: "Allows the owner to distribute..." yet implements no ownership check
2. Contest specification requires owner-only access for distribution functions

## Impact

The owner's ability to manage distribution timing is completely compromised. Token distributions trigger NFT minting with vesting schedules, and the owner may need to:

- Align distributions with marketing campaigns or token launches
- Time distributions based on market conditions
- Ensure proper infrastructure is ready to support newly vested tokens

The codebase shows `distributeRemainingRewardTVS()` correctly uses `onlyOwner`, creating a confusing security model where some distribution paths are protected while others are wide open. 

## Recommended Mitigation

Apply the `onlyOwner` modifier to enforce owner-only access
  