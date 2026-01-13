# [000320] Pool-count guard allows 11 pools instead of 10 in `AlignerzVesting`
  
  ### Description:
The [`createPool()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L679-L702) function in `AlignerzVesting` is intended to limit the number of pools per bidding project, but the guard condition is off-by-one. The current logic is:

```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());

uint256 poolId = biddingProject.poolCount;
...
biddingProject.poolCount++;
```

When `biddingProject.poolCount` is `10`, the `require` condition still passes, and `poolCount` is incremented to `11`, resulting in a project with 11 pools (`poolId` values 0–10). While the rest of the contract correctly uses `poolCount` as the number of pools and does not break, this behavior exceeds the intended “ten pools per project” limit and can contradict assumptions made by off-chain components or documentation.

### Impact:
A project can end up with 11 pools instead of the intended maximum of 10, potentially breaking off-chain assumptions, though without direct fund risk.

### Recommendation:
Tighten the guard condition so that the 11th pool cannot be created, for example:

```solidity
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

This enforces a strict maximum of 10 pools per project while keeping the rest of the logic unchanged.

  