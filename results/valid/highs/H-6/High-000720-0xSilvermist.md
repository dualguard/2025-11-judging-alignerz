# [000720] Using `getTotalMinted` in `_setDividends` causes inevitable DoS
  
  ### Summary

The `_setDividends` function loops through all ever-minted NFTs (including burned ones) using `getTotalMinted()`. As the protocol grows and more NFTs are minted over time, this loop will inevitably exceed block gas limits, permanently breaking the dividend distribution system.

### Root Cause

In the `_setDividends` function in `A26ZDividendDistributor.sol`, the loop iterates over `nft.getTotalMinted()` which returns the total count of all NFTs ever minted, including those that have been burned. This creates an unbounded loop that grows indefinitely with protocol usage and never decreases even when NFTs are burned.

```solidity
function _setDividends() internal {
->    uint256 len = nft.getTotalMinted();
    
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```

The same applies to the `getTotalUnclaimedAmounts`.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. **Early stage (100 total mints):**
   - Owner calls `setDividends()` → loops 100 times → succeeds 

2. **Growth stage (1,000 total mints, 300 burned):**
   - Owner calls `setDividends()` → loops 1,000 times (not 700!) → still works 

3. **Mature stage (5,000 total mints, 2,000 burned):**
   - Owner calls `setDividends()` → loops 5,000 times (not 3,000!) → approaching gas limit 

4. **Breaking point (8,000+ total mints):**
   - Owner calls `setDividends()` → loops 8,000+ times → out of gas 
   - Function permanently broken, no dividends can ever be distributed again

### Impact

`setDividends` becomes permanently unusable as the protocol grows.

### PoC

_No response_

### Mitigation

Use `totalSupply()` instead of `getTotalMinted()` in `_setDividends` and  `getTotalUnclaimedAmounts` because `totalSupply()` function correctly accounts for burned tokens:

```diff
function _setDividends() internal {
-   uint256 len = nft.getTotalMinted();
+   uint256 len = nft.totalSupply();
    
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```

  