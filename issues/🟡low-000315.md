# [000315] Pause on AlignerzNFT does not stop transfers, weakening incident response controls
  
  ### Description:
The [`AlignerzNFT`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol) contract is documented as using `pause()` to halt “minting and transfers”, but the implementation only guards `mint()` and `burn()` while leaving token transfers fully functional.

Key parts:

```solidity
contract AlignerzNFT is Ownable, ERC721A, Pausable {
    // ...

    function pause() external virtual onlyOwner {
        _pause();
    }

    function unpause() external virtual onlyOwner {
        _unpause();
    }

    function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
        // ...
        _safeMint(to, 1);
    }

    function burn(uint256 tokenId) external whenNotPaused onlyMinter {
        require(_exists(tokenId), "ALIGNERZ: Token Id does not exist");
        _burn(tokenId);
    }
}
```

The inherited ERC721A `transferFrom()` / `safeTransferFrom()` are not overridden to check the paused state, nor is `_beforeTokenTransfers()` used to enforce `whenNotPaused`. As a result:

- Calling `pause()` prevents new mints and burns.
- Existing NFTs can still be freely transferred between addresses.

This contradicts the intended behavior implied by the documentation (“pauses the contract (minting and transfers)”) and weakens the emergency control surface during an incident, since operators may believe NFTs (TVSs) are frozen while they remain movable on secondary markets.

### Impact:
When `pause()` is activated, existing NFTs remain transferable, reducing the effectiveness of pausing as an incident-response / containment tool.

### Recommendation:
Enforce the paused state on transfers as well. For example, override `_beforeTokenTransfers()` in `AlignerzNFT` to revert when `paused()` is true for any non-mint/non-burn operation, or override `transferFrom()` / `safeTransferFrom()` to add `whenNotPaused`. This ensures `pause()` truly freezes minting, burning, and transfers as suggested by the documentation.

  