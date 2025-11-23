# [000131] `calculateFeeAndNewAmountForOneTVS()` loop doesn't increment `i` causing an infinite loop
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS()` helper in `FeesManager` contains an unincremented loop index. When called with a non-zero length, the function enters an infinite loop. This helper is used by `mergeTVS()` and `splitTVS()` in `AlignerzVesting`, breaking both core functionalities

### Root Cause

The fee helper is defined in `FeesManager.sol` as:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

- The loop is declared as `for (uint256 i; i < length;)` with no increment expression in the header and no `++i` in the body. When `length > 0`, the loop condition `i < length` is true for `i = 0` and remains true forever because `i` is never incremented, creating an infinite loop and eventual out-of-gas revert.

Because both `mergeTVS()` and `splitTVS()` call `calculateFeeAndNewAmountForOneTVS()`, both functions will revert with an out-of-gas error because of the infinite loop. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

The issue has a medium severity because it causes a  DOS for two core vesting operations (`mergeTVS()` and `splitTVS()`) on all non-trivial TVSs. Users cannot execute merges or splits at all.

### PoC

_No response_

### Mitigation

Simply increment i at the end of the loop.
  