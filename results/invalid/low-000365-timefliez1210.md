# [000365] L01 Merge does not check for duplicate NFTs to be passed, potentially locking user funds
  
  ## Summary

The `AlignerzVesting::mergeTVS` function allows users to include the target NFT (`mergedNftId`) in the list of source NFTs (`nftIds`) to be merged. If this happens, the contract burns the target NFT while updating its storage. Since `claimTokens` requires the caller to own the NFT (via `nftContract.extOwnerOf`), and the NFT is burned, the user permanently loses access to the merged tokens.

## Root Cause

In `AlignerzVesting::mergeTVS`, the code iterates over `nftIds` and calls `_merge` for each:

```solidity
        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
```

In `_merge`:

```solidity
    function _merge(..., uint256 nftId, ...) internal ... {
        // ...
        nftContract.burn(nftId); // <--- Burns the NFT
    }
```

If `nftId == mergedNftId`, the function correctly calculates the math (adding the amount to itself, minus fee), but then **burns the NFT**. The `Allocation` struct remains in storage with the new balance, but the "Key" (the NFT) is destroyed.

## Internal Pre-Conditions

1.  User owns an NFT.

## External Pre-Conditions

1.  User calls `mergeTVS` and accidentally includes the target NFT ID in the `nftIds` array.

## Attack Path

Just a bug.

## Impact

**Low/Info**. Permanent loss of funds due to a likely user error or frontend integration mistake. The protocol should prevent this invalid state on the protocol level.

## PoC

Trivial.

## Mitigation

Add a check in `mergeTVS` or `_merge` to ensure `nftId != mergedNftId`.

```solidity
        for (uint256 i; i < nbOfNFTs; i++) {
            require(nftIds[i] != mergedNftId, "Cannot merge NFT into itself");
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
```

  