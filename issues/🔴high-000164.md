# [000164] User can brick the `A26ZDividendDistributor` contract by slitting NFTs
  
  ### Summary

Imagine the following scenario:
1. User has a small NFT which practically doesn't give him any benefits because it has too small of a value
2. The user decides to split it in multiple NFTs , aiming to brick the [`A26ZDividendDistributor`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L9) contract

This `A26ZDividendDistributor` have 2 points which may be vulnarable to this attack, which leads to almost inevitable success. The first point is the itterations through the NFTs in the `getTotalUnclaimedAmounts` function and the second is the iteration through the NFTs in the `_setDividends` function. Both are points of failure  and lead to almost guaranteed success in this DoS attack

### Root Cause

The two points of failure in the mentioned functions

### Internal Pre-conditions

User has small value NFT

### External Pre-conditions

None

### Attack Path

1. User splits his small NFT on multiple small ones. It is possible for the new NFTs to have value of 0, due to the way the `splitTVS` function works
2. The `A26ZDividendDistributor` contract gets DoSed by the large number of NFTs it need to iterate through

### Impact

DoS of the `A26ZDividendDistributor` contract

### PoC

none

### Mitigation

Limit the user's ability to split NFTs
  