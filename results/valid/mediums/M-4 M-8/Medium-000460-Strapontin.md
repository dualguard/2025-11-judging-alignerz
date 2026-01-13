# [000460] Last NFT owner will not be able to receive dividends
  
  ### Summary

The contract to distribute dividends eventually call `_setDividends` that will loop over all NFTs minted in order to give dividends to their owners. However, since the loop starts at `i = 0`, and the first nft starts at index `1`, the loop will stop before setting the dividend for the last NFT, so their owner will not receive anything.

Note that the function `getTotalUnclaimedAmounts` also has this issue.

### Root Cause

`i` should start at `nft._startTokenId()`

### Internal Pre-conditions

At least 1 NFT is minted

### External Pre-conditions

None

### Attack Path

No sophisticated attack path necessary. The issue will occur when dividends are set for users.

### Impact

Every time dividends will be given, the last NFT holder will not be able to receive anything.

### Mitigation

Update the loops to start at `_startTokenId`

```diff
-   for (uint i; i < len;) {
+   for (uint i = nft._startTokenId(); i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
```
  