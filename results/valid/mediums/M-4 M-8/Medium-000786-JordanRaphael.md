# [000786] Last NFT ID Excluded from Dividend Distribution
  
  ### Summary

An off-by-one error will cause the last NFT holder to lose dividend distributions as the protocol will skip the final NFT ID during dividend calculations due to incorrect loop bounds.

### Root Cause

In `A26ZDividendDistributor.sol` the `_setDividends()` and `getTotalUnclaimedAmounts()` functions use an incorrect loop boundary. Since NFT IDs are 1-indexed (starting from 1), iterating with `i < len` where `len` equals `getTotalMinted()` will iterate from 0 to len-1, causing the loop to never process the last NFT ID. The correct boundary should be `i < len + 1` to account for all minted NFTs.

Example: If `getTotalMinted()` returns 100, NFT IDs 1-100 exist, but the loop only processes indices 0-99, missing the holder of NFT ID 100.

[getTotalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136)

[_setDividends](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls `setDividends()` to distribute dividends to all NFT holders
2. The function iterates from index 0 to len-1, skipping index len (the last NFT ID)
3. `getTotalUnclaimedAmounts()` is called to calculate total unclaimed amounts, using the same flawed loop
4. Due to the mismatch in what dividends are calculated versus what amounts were counted, the last NFT holder receives no dividend payout while their unclaimed amount was never included in the total


### Impact

The holder of the last minted NFT suffers a loss equal to their entitled dividend share. The exact loss depends on the stablecoin amount being distributed and the unclaimed amount assigned to the last NFT ID. This is a recurring loss on every dividend distribution cycle.


### PoC

_No response_

### Mitigation

Change the loop condition in both `_setDividends()` and `getTotalUnclaimedAmounts()` from `i < len` to `i <= len`, in order to include all the NFT ids in the calculations.
  