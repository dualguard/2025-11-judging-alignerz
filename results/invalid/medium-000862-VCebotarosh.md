# [000862] Missing onlyOwner Modifier Allows Users to Bypass RewardProject Deadline Enforcement
  
  ### Summary

The `deadline` parameter in RewardProject struct represents the "deadline after which it's impossible for users to claim TVS or refund". While comments indicate that `distributeRewardTVS()` and `distributeStablecoinAllocation()` are intended to be owner-restricted, both functions lack the onlyOwner modifier. As a result, the user can claim their rewards even after the deadline.

### Root Cause

The `distributeRewardTVS()` and `distributeStablecoinAllocation()` functions omit the `onlyOwner` modifier despite documentation indicating they should be restricted. Because these functions perform an internal deadline check and revert only for pre-deadline calls, external users can invoke them freely after the deadline has passed. This design flaw unintentionally grants users the ability to claim rewards beyond the intended expiry period, bypassing the logic associated with deadline.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L527

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L542


### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

Users can claim TVS or stablecoin rewards after the intended claim period has expired, rendering the deadline parameter ineffective.

### PoC

None

### Mitigation

Add the onlyOwner modifier to both `distributeRewardTVS()` and `distributeStablecoinAllocation()` to ensure they can only be executed by the contract owner.
  