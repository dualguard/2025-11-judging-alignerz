# [000819] ERC721A starts token IDs from 1 rather than from 0 as per the code comments
  
  Per the NatSpec of `ERC721A.sol`:

```solidity
/**
 * Assumes serials are sequentially minted starting at _startTokenId() (defaults to 0, e.g. 0, 1, 2, 3..).
 **/
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/ERC721A.sol#L33

However, `_startTokenId()` is hardcoded to 1 rather than 0

```solidity
function _startTokenId() internal pure virtual returns (uint256) {
    return 1;
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/ERC721A.sol#L102-L107

`AlignerzNFT` makes no override on the start token ID, and minting also always mints sequentially.

This makes NFT IDs sequential starting from 1 rather than from 0, deviating from the code comments.

  