# [000059] Inverted Token Check Zeroes Dividends for Correct TVS Holders
  
  ## Summary

The [`getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) function in `A26ZDividendDistributor` uses an inverted token comparison check. It returns 0 (no dividends) when the NFT's token matches the distributor's token, and calculates dividends when tokens don't match. This causes the dividend system to reward the wrong TVS holders - those holding different tokens instead of the intended $A26Z token holders.

## Vulnerability Details

### The Bug

The dividend distributor is designed to reward TVS holders of a specific token (e.g., $A26Z) with stablecoin dividends based on their unclaimed token amounts. The `getUnclaimedAmounts()` function should calculate dividends only for NFTs that hold the correct token.

However, the token check is inverted:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  // <- wrong
}
```

## Impact

### Complete Dividend Misdirection

The dividend distribution system is completely broken:

1. Intended recipients get nothing: TVS holders of the target token (e.g., $A26Z) receive 0 dividends despite being the intended beneficiaries
2. Wrong recipients get rewards: TVS holders of other tokens receive dividends they shouldn't get
3. Protocol loses funds: Stablecoins are distributed to the wrong users and cannot be recovered

## Severity


## Recommended Mitigation

Invert the comparison operator:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // Only calculate dividends for NFTs holding the correct token
    if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;  // <- fix this
    // ...
}
```

  