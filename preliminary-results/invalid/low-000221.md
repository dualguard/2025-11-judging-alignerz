# [000221] Pool creation allows more pools instead of the intended maximum of 10
  
  ### Summary

A flawed boundary check in [`createPool()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689) will cause an off-by-one error for pool creation, allowing the owner to create 11 pools instead of 10. Although the function is restricted, this violates the intended protocol configuration and may break assumptions in off-chain logic or UI components.

### Root Cause

In `createPool()` (AlignerzVesting.sol), the check:

```js
require(biddingProject.poolCount <= 10);
```

allows `poolCount == 10` to pass, enabling creation of an 11th pool before reverting on the next call.

This is an off-by-one error; the intended maximum is 10 pools (indexes 0–9), but the condition permits index 10 as well.

### Internal Pre-conditions

1. The owner must call createPool() repeatedly until poolCount == 10.
2. The project must not be closed.
3. The project ID must be valid.

### External Pre-conditions

None.

### Attack Path

None.

### Impact

The protocol supports one additional pool beyond the intended maximum, which may cause inconsistencies between smart-contract constraints, business logic, documentation, and any off-chain systems that assume a strict limit of 10.

### PoC

_No response_

### Mitigation

Replace the check with:

```js
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

This enforces a strict upper bound of 10 pools (IDs 0–9).
  