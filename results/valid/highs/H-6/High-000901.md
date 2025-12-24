# [000901] Denial-of-Service via Unbounded Iteration in _setAmounts and _setDividends
  
  ### Summary

When setting dividends via `setUpTheDividends` the contract calls:

- `_setAmounts()` — computes `totalUnclaimedAmounts`
- `_setDividends()` — allocates `stablecoins` proportionally to each NFT holder

Both functions iterate over **all minted NFTs** using:

```solidity
uint256 len = nft.getTotalMinted();
for (uint256 i; i < len;) { ... }
```

This design creates an **unbounded loop whose gas consumption grows linearly with the number of NFTs**. Once the NFT count becomes sufficiently large, these loops will **consume more gas than is available in a single transaction**, making the functions revert permanently.

### Root Cause

[`_setAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211) loops over every NFT

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
}
```

Inside it, [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) does:

```solidity
uint256 len = nft.getTotalMinted();
for (uint256 i; i < len;) {
    (, bool isOwned) = safeOwnerOf(i);
    if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
    ++i;
}
```

 [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L225) loops over every NFT again

```solidity
uint256 len = nft.getTotalMinted();
for (uint256 i; i < len;) {
    (address owner, bool isOwned) = safeOwnerOf(i);
    if (isOwned) {
        dividendsOf[owner].amount += (
            unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
        );
    }
    ++i;
}
```

### Internal Pre-conditions

The protocol needs to grow to a larger size for alot of NFTs to be minted, this is not unprecedented, as it is likely that even 100 NFT's which is not significant can brick the distributor.

### External Pre-conditions

N/A

### Attack Path

1. Protocol grows in size, users invest TVS' are minted to investors
2. Protocol decides to distribute rewards to investors, but it reverts due to DOS

### Impact

Since gas consumption grows linearly with NFT count, it will eventually exceed the block gas limit. This is very likely, and hurts users who invest into the protocol.

### PoC

N/A

### Mitigation

N/A
  