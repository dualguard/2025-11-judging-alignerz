# [000622] Owner will be unable to distribute dividends due to out-of-gas errors when NFT count grows large from splits
  
  ### Summary

Nested unbounded loops in `setUpTheDividends` will cause out-of-gas reverts when processing large numbers of NFTs, as the function iterates through all minted NFTs multiple times with nested loops, and since users can split NFTs indefinitely, the total NFT count can grow to thousands or tens of thousands, making quarterly dividend distribution impossible.

### Root Cause

In [A26ZDividendDistributor.sol:setUpTheDividends](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111C5-L114C6), the function calls `_setAmounts()` and `_setDividends()`, both of which have unbounded loops:
```solidity
function setUpTheDividends() external onlyOwner {
    _setAmounts();    // ← Loop 1 (has two loops inside)
    _setDividends();  // ← Loop 2
}
```
_setAmounts -> getTotalUnclaimedAmounts
```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();  // Could be 10,000+
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);  // ← Nested loop!
        unchecked {
            ++i;
        }
    }
}
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Users and KOLs will have NFTs, plus users can split NFTs as much as they want. This can naturally give a large amount of NFTs, and looping 3 times over those NFTs will result in a DOS
Plus, a malicious user can intentionally split their NFT into thousands to break the dividend distribution

### Impact

Dividend distribution becomes impossible

### PoC

_No response_

### Mitigation

Do Batch Processing 
  