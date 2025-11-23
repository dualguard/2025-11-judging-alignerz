# [000140] Missing access control for `distributeRewardTVS` and `distributeStablecoinAllocation`
  
  ### Summary

The `AlignerzVesting` contract exposes `distributeRewardTVS()` and `distributeStablecoinAllocation()` as external entry points for post‑deadline reward distribution, while their NatSpec comments describe them as owner-only operations. Both functions are declared `external` without any access-control modifier, allowing any address to trigger these distributions after the claim window closes.

### Root Cause

.

### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

.

### Impact

The severity is low because the distributions still go to legitimate KOLs and do not enable theft, but they weaken operational control and can cause avoidable transaction failures.

### PoC

_No response_

### Mitigation

Add the onlyOwner modifier to both functions.
  