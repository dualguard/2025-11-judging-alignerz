# [000918] provided price in `createPool` are not being used
  
  ### Summary

the token price set for the pool when creating one is provided as parameter but it is not being used anywhere beside an emitted event.

### Root Cause

[AlignerzVesting.sol#L678-L679](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L678-L679)
```solidity
    /// @param tokenPrice token price set for this pool
    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
```

`tokenPrice` is provided in the `createPool` function but it is only emitted and not being set or used for anything

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. owner create pool with the expectation price being set to 100
2. but it does not do anything

### Impact

broken core function because the price is not set

### PoC

_No response_

### Mitigation

remove it completely from the params or use the price inside the function.
  