# [000306] _setDividends() and getTotalUnclaimedAmounts() function in A26ZDividenDistributor iterate NFT from index 0 instead of 1, even though _startTokenId is 1
  
  ### Summary

`_setDividends()` **and `getTotalUnclaimedAmounts()` function in `A26ZDividenDistributor` iterate NFT from index 0 instead of 1, even though `_startTokenId()` is 1.

`_setDividends()` :

```solidity
function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) { 
        --- rest of code ---
}
```

`getTotalUnclaimedAmounts()` :

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) { 
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) { 
        --- rest of code ---
}
```

`_startTokenId()` :

```solidity
function _startTokenId() internal pure virtual returns (uint256) {
        return 1;  
}
```

This lead to use more gas when iterating and increase chance of OOG and the application is wrong because the first NFT to be minted has an Id starting from 1 and NFTs that have Id = 0 are never minted, so iteration from 0 is useless.

### Root Cause

In [A26ZDividendDistributor:128](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128) & [215](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L215) start iterating from 0

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

This will directly execute when `_setDividends()` and `getTotalUnclaimedAmounts()` got called

### Impact

This lead to use more gas when iterating and increase chance of OOG and the application is wrong because the first NFT to be minted has an Id starting from 1 and NFTs that have Id = 0 are never minted, so iteration from 0 is useless.

### PoC

_No response_

### Mitigation

_No response_
  