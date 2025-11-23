# [000011] Protocol will misallocate dividends for the last TVS NFT holder
  
  ### Summary

An off-by-one error when iterating over TVS NFT IDs in [`A26ZDividendDistributor::getTotalUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) and [`A26ZDividendDistributor::_setDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224) will cause permanent misallocation of dividends for the last TVS NFT holder as the protocol will skip that holder when computing total unclaimed amounts and when setting `dividendsOf`, distributing their share to other holders instead.

### Root Cause

In `A26ZDividendDistributor::getTotalUnclaimedAmounts` and `A26ZDividendDistributor::_setDividends` the contract uses:

```solidity
uint256 len = nft.getTotalMinted();
for (uint256 i; i < len; ) {
    // ...
}
``` 
to iterate TVS NFTs even though NFT IDs start from `1` and the last minted ID equals `len`, so the loop processes IDs `[0, len-1]`, skips ID `len` entirely, and never includes the last TVS holder in `getTotalUnclaimedAmounts()` or `_setDividends()`.

You can see that in [`ERC721A`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L105-L107), `startTokenId` is 1

### Internal Pre-conditions

1. At least one TVS NFT has been minted (so `nft.getTotalMinted() > 0`).
2. The owner calls `setUpTheDividends()` or `setAmounts()` followed by `setDividends()` to initialize and distribute dividends.

### External Pre-conditions

None

### Attack Path

1. Owner calls `setUpTheDividends()` (or `setAmounts()` then `setDividends()`) to distribute dividends to all TVS holders.
2. `getTotalUnclaimedAmounts()` reads `len = nft.getTotalMinted()` and loops `for (uint i; i < len; )`, only considering IDs `0..len-1`, so the last NFT `len` is never included in `_totalUnclaimedAmounts`.
3. `_setDividends()` uses the same loop bounds and therefore never updates `dividendsOf[owner]` for the holder of NFT ID `len`.
4. When the last TVS holder later calls `claimDividends()`, `dividendsOf[user].amount` is zero, so they receive no dividends while the full `stablecoinAmountToDistribute` has already been allocated among the other holders.

### Impact

The last TVS NFT holder suffers a complete loss of their dividend share for each distribution, while other TVS holders receive a proportionally larger share of `stablecoinAmountToDistribute`; in the worst case (even allocations), roughly `1 / totalNFTs` of every dividend pool is misallocated from the last TVS holder to the rest of the holders, and if the last holder has a larger allocation their loss can be significantly greater.

### PoC

_No response_

### Mitigation

Change the loop bounds in both `getTotalUnclaimedAmounts()` and `_setDividends()` to iterate over the actual minted ID range, e.g. 
```solidity
for (uint256 i = 1; i <= len; ) { ... unchecked { ++i; } }
``` 
or otherwise ensure that the iteration starts from the correct first token ID and includes the last minted ID returned by `getTotalMinted()`.
  