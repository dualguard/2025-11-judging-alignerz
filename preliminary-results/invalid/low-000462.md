# [000462] Stablecoin allocations cannot be claimed or distributed when block.timestamp is exactly equal to `rewardProject.claimDeadline`
  
  ### Summary

Stablecoin allocations cannot be claimed or distributed when block.timestamp is exactly equal to `rewardProject.claimDeadline`.

### Root Cause

The logic enforcing claimability requires that distribution begins after the `rewardProject.claimDeadline`, rather than at or after it. As a result, the system does not allow actions to proceed when `block.timestamp` matches the deadline precisely.

### Attack Path

1. The current block timestamp is exactly equal to `rewardProject.claimDeadline`.
2. A user attempts to call `distributeStablecoinAllocation` or `claimStablecoinAllocation`.
3. The call reverts due to the strict timestamp check.

### Impact

The transaction reverts, preventing users from claiming or receiving their stablecoin allocations at the intended time, resulting in an inconvenient and unexpected user experience.
  