# [000536] getTotalUnclaimedAmounts() and _setDividends() have incorrect bounds for nft Ids.
  
  ## Summary

The functions `getTotalUnclaimedAmounts()` and `_setDividends()` in the for-loop have incorrect bounds when enumerating NFT IDs. This leads to incorrect calculations of dividends.



## Root cause

Let's consider both functions. We can see that in for-loops, the index starts from 0 and ends at `nft.getTotalMinted()`.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129

```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
@=>        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216

```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
@=>        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```


But NFT ID starts from 1 (https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L106), and also the last NFT ID will be equal to `nft.getTotalMinted()` (https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L127).



## Impact

These functions just don't work as they should and produce incorrect results.



## Mitigation

```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
-        for (uint i; i < len;) {
+        for (uint i=1; i <= len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```



```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
-        for (uint i; i < len;) {
+        for (uint i=1; i <= len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

  