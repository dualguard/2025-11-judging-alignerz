# [000869] `_setDividends()` and `getTotalUnclaimedAmounts()` iterate over `getTotalMinted()` indefinitely, which eventually will lead to OOG on each call
  
  ### Summary

`_setDividends()`  and `getTotalUnclaimedAmounts ()` iterate over all NFT indexes ever minted:
```solidity
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
```
Regardless of whether they were not burned. Eventually ( by itself over time or due to an attack ), this will make that function OOG on each attempt to set new dividends, breaking core functionality. 

### Root Cause

Loop over arbitrarily large number equal to the total amount of NFTs every minted on AlignerzNFT contract.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

Malicious user can deliberately use the split functionality to create a huge amount of NFTs - enough to cause OOG on `_setDividends()` and `getTotalUnclaimedAmounts()` 

### Impact

Owner cannot set dividends or amounts.

### PoC

_No response_

### Mitigation

_No response_
  