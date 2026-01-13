# [000218] Unbounded Loops in Dividend Setup May Cause DoS of Dividend Distribution
  
  ### Summary

The dividend distribution process requires iterating over all minted TVS NFTs twice in a single call. As the number of NFTs may grow large, these loops can exceed the block gas limit, causing `A26ZDividendDistributor::setUpTheDividends()` to revert.

### Root Cause

Both operations used during dividend setup [`A26ZDividendDistributor::setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L110-L114) perform unbounded full iterations over all minted NFTs (bidding + rewards projects) `nft.getTotalMinted()`:

* `_setAmounts()` → calls `getTotalUnclaimedAmounts()` which loops over all NFTs.
* `_setDividends()` → loops again over all NFTs.

While on megaETH the DoS is not feasible due to its micro-block execution model that automatically splits long loops, on Polygon and Arbitrum this issue is feasible because both chains enforce a per-block gas limit (e.g. Polygon - 45M gas units). As the number of minted NFTs grows, the unbounded loops inside `setUpTheDividends()` may eventually exceed those limits.

Although the owner can call `setAmounts()` and `setDividends()` separately, both functions independently iterate over the entire NFT supply (nft.getTotalMinted()), meaning the DoS risk applies to each function individually, not only when they are called together via `setUpTheDividends()`.

### Internal Pre-conditions

1. Many NFTs exist (from bidding flows and reward flows).
2. Admin calls the required function setUpTheDividends().

### External Pre-conditions

None.

### Attack Path

None.

### Impact

DoS of dividend setup if NFTs count grows too large before dividend setup.

### PoC

None.

### Mitigation

One solution would be to use batched or paginated dividend setup.
  