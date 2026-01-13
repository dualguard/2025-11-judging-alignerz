# [000816] Missing `onlyOwner` on `distributeRewardTVS()` and `distributeStablecoinAllocation()`, contrary to natspec
  
  ### Summary

Missing `onlyOwner` modifier on `distributeRewardTVS()` and `distributeStablecoinAllocation()` will cause loss of owner control. KOLs can distribute rewards to themselves without owner approval after the deadline.

### Root Cause


`distributeRewardTVS()` and `distributeStablecoinAllocation()` are documented to be owner-only functions but lack the `onlyOwner` modifier, allowing anyone to call them after the deadline.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L537

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L522

```solidity
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
function distributeRewardTVS(...) external { // missing onlyOwner
    //...
    require(
        block.timestamp > rewardProject.claimDeadline,
        Deadline_Has_Not_Passed()
    );
    //...
}

/// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
function distributeStablecoinAllocation(...) external { // missing onlyOwner
    //...
    require(
        block.timestamp > rewardProject.claimDeadline,
        Deadline_Has_Not_Passed()
    );
    //...
}
```

### Internal Pre-conditions

1. At least one KOL does not claim their reward before deadline.

### External Pre-conditions

None

### Attack Path


1. Project owner sets TVS or stablecoin allocations for KOLs through `setTVSAllocation()` or `setStablecoinAllocation()`.
2. During claim window, KOL don't claim rewards.
3. Deadline passes, owner should have complete power to distribute rewards or not.
4. Instead of waiting for owner decision, KOL calls `distributeRewardTVS()` or `distributeStablecoinAllocation()` 

### Impact

The owner loses access control over reward distribution after the deadline.


### PoC

_No response_

### Mitigation

Add `onlyOwner` modifier to both functions `distributeRewardTVS()`, `distributeStablecoinAllocation()`.

  