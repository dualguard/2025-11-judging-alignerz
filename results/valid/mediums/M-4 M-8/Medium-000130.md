# [000130] Off-by-One Loop Error in Dividend Distribution
  
  ### Summary

The dividend calculation functions (`A26ZDividendDistributor::getTotalUnclaimedAmounts()`; `A26ZDividendDistributor::_setDividends()`) have an off-by-one error that systematically skips the last minted NFT. 


### Root Cause

Since that TokenId is incremented starting at 1, while the loop is incremented starting at 0. The use of `safeOwnerOf` leads to the first invalid NFT (0) being skipped, stopping the call from reverting, but the last NFT still never gets used in the loop. This leads to a loss of funds for users.

- `_startTokenId()` returns `1` (not 0)
- Valid token IDs: `1, 2, 3, ..., n`
- `_totalMinted()` returns `_currentIndex - _startTokenId()` 
- If `getTotalMinted()` returns `n`, valid IDs are `1` through `n`

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L220

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L146-L157

- `i = 0`: Checks invalid tokenId 0 (skipped by `safeOwnerOf`)
- `i = 1` to `n-1`: Checks tokenIds 1 through n-1
- `i = n`: is never reached

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

- Last minted NFT owner is excluded from dividend calculations, which leads to `totalUnclaimedAmounts` being underreported (missing the last NFT's unclaimed balance). `dividendsOf[owner]` for the last NFT holder is never populated, the last NFT owner receives zero dividends permanently

This leads to a loss of yield for the owner of the last NFT

### PoC

_No response_

### Mitigation

Change loop to iterate from 1 to `len` (inclusive)
  