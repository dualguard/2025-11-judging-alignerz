# [000244] Division by Zero Causes DOS in `A26ZDividendDistributor::_setDividends()` When No Unclaimed Amounts Exist
  
  - **Description**

The `::_setDividends()` function performs division by `totalUnclaimedAmounts` without validating that it is non-zero. When all NFTs are fully vested or no NFTs exist, this valud becomes zero, causing the function to revert. This prevents the owner from distributing dividends even when stablecoins have been deposited.

- **Impact**

- Dividend distribution is blocked until new NFTs with unclaimed amounts exist
- Funds are stuck (although recoverable via `::withdrawStuckTokens()`)

- **Recommended Mitigation**

Add a validation check before performing division:

```diff
function _setDividends() internal {
+   require(totalUnclaimedAmounts > 0, "No unclaimed amounts to distribute");
    
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount += ((unclaimedAmountsIn[i] * stablecoinAmountToDistribute)
                    / totalUnclaimedAmounts);
        }
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```
  