# [000028] `getTotalUnclaimedAmounts()` iterates over the entire NFT supply, causing OOG and blocking dividend setup
  
  ### Summary

The function `getTotalUnclaimedAmounts()` computes the total unclaimed TVS amounts by looping over **all minted NFTs**:

```solidity
uint256 len = nft.getTotalMinted();
for (uint i; i < len; ) {
    (, bool isOwned) = safeOwnerOf(i);
    if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
    unchecked { ++i; }
}
```

This requires an **unbounded full scan over the entire NFT collection**, regardless of which token type the distributor is interested in.

As the number of minted NFTs grows, this loop becomes increasingly gas-expensive and will eventually **run out of gas**, preventing the owner from calling `setUpTheDividends()`, `setAmounts()`, or updating dividend distributions. This permanently blocks dividend updates and renders the contract non-functional.

### Root Cause

`getTotalUnclaimedAmounts()` loops over all the minted NFTs, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L130.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Dividend setup becomes impossible once the NFT count grows beyond a certain point, causing the dividend system to become permanently unusable on mainnet.

### PoC

_No response_

### Mitigation

Consider allowing the owner to **batch process NFT IDs** by passing a list of relevant IDs, instead of scanning the entire NFT range.
  