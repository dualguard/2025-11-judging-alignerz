# [000067] Off-By-One Error Allows Creation of 11 Pools
  
  ## Summary

The [`createPool()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689) function contains an off-by-one error in its pool limit check. The function uses `<=` instead of `<` when checking if the pool count exceeds 10, allowing 11 pools to be created instead of the intended maximum of 10.

## Vulnerability Details

### Root Cause

The check uses `poolCount <= 10` which allows pool counts of 0 through 10, resulting in 11 pools:

```solidity
function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
    external
    onlyOwner
{

    // ...
    // bug: allows poolCount = 10, which creates pool id 10 (the 11th pool)
    require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());

    uint256 poolId = biddingProject.poolCount;  // poolId can be 0-10 (11 pools)
    // ...
}
```

## Impact

### Design Limit Exceeded

The error message states "Cannot_Exceed_Ten_Pools_Per_Project", indicating the intended limit is 10 pools. However, the off-by-one error permits 11 pools, which may:

- Exceed the stated design limit
- Cause confusion in off-chain systems expecting max 10 pools
- Break assumptions in other parts of the codebase

## Recommended Mitigation

Change the check to use `<` instead of `<=`

  