# [000822] `A26ZDividendDistributor`: `getTotalUnclaimedAmounts()` and `_setDividends()` wrongly loops 0-indexed, but the NFT IDs are 1-indexed
  
  ### Summary

`getTotalUnclaimedAmounts()` and `_setDividends()` wrongly assumes NFTs are 0-indexed, but the NFT IDs are 1-indexed. This causes the last NFT to be wrongly excluded from rewards calculations.

### Root Cause


In `ERC721A`, NFTs are minted starting from ID 1, and sequentially increasing:

```solidity
function _startTokenId() internal pure virtual returns (uint256) {
    return 1;
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/ERC721A.sol#L105-L107

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol#L106

That means the NFTs are 1-indexed, i.e. the NFTs minted are of IDs 1, 2, 3, 4, ... up to the total number of minted NFTs, inclusive.

However, in `A26ZDividendDistributor`, there are some functions that loops over all NFT IDs, namely `getTotalUnclaimedAmounts()` and `_setDividends()`. These function loops through the NFTs starting from ID 0 instead of 1:

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) { // @audit wrongly loops from 0 instead of 1, and excludes the last NFT
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}
``` 

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216

This causes the most recent NFT minted to be never looked at, and will never have its reward distributed to.

### Internal Pre-conditions

The final NFT must be that of an A26Z TVS. This is fairly easily possible by, for example, someone splitting an A26Z TVS, causing the most recent NFT minted to fit the criteria.

### External Pre-conditions

None

### Attack Path

- Admin calls `setUpTheDividends()` to set up profit distribution.
- The contract wrongly excludes the final NFT, causing its holder to lose the rightful rewards.

### Impact

The most recent NFT minted will not get its share of rewards.

### PoC

_No response_

### Mitigation

Loop from 1 instead, up to **and including** len.
  