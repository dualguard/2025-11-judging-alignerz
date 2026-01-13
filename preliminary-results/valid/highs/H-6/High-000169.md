# [000169] Dividents will be bricked over time when NFTs are minted
  
  ### Summary

When more NFTs are minted, the gas cost for a transaction will increase, until the block gas limit is exceeded and the transaction reverts. This is the case with the [`A26ZDividendDistributor::getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127) function. When more and more users mint NFTs and the system develops, the function will eventually start to exceed the gas limit at some point no matter if the NFTs are burned or not (because the burned NFTs are part of the iterations as well). This will lead to full DoS of the dividend distribution.

### Root Cause

Root cause of the issue is the practically infinite number of NFTs that can be minted

### Internal Pre-conditions

Everything goes smoothly. Users interact with the system and mint NFTs through projects etc.

### External Pre-conditions

None

### Attack Path

1. Everything goes smoothly. Users interact with the system and mint NFTs through projects etc.
2. The `getTotalUnclaimedAmounts` function exceeds the gas limits of the different chains, leading to impossibility to distribute the dividiends

### Impact

Full DoS of the `A26ZDividendDistributor` contract

### PoC

-

### Mitigation

Implement functionality which allows the owner to account for all the NFTs in portions. For example 50 NFTs per transaction. This will allow him to distribute everything and even though it is not in 1 tx the end result will be distributed funds.
  