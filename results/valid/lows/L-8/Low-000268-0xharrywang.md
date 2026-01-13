# [000268] NFT holders can still transfer NFTs while `AlignerzNFT` is paused
  
  ### Summary

When the `AlignerzNFT` contract is in a paused state, NFT holders can still use `transferFrom()` and `safeTransferFrom()` to transfer their NFTs. This behavior conflicts with the stated intention that pausing should stop both minting and transfers.

### Root Cause

In `AlignerzNFT.sol`, the NatSpec comment clearly states that `pause()` is intended to pause minting and transfers:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80-L83
```solidity
    /// @notice pauses the contract (minting and transfers)
    function pause() external virtual onlyOwner {
        _pause();
    }
```

However, in the underlying `ERC721A` implementation, both `transferFrom()` and `safeTransferFrom()` are not guarded by any `whenNotPaused` check and remain callable even when the contract is paused:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L294-L306
```solidity
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        _transfer(from, to, tokenId);
    }
    
	function safeTransferFrom(address from, address to, uint256 tokenId) public virtual override {
        safeTransferFrom(from, to, tokenId, "");
    }
```


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The owner calls `pause()`, and the `AlignerzNFT` contract enters the paused state.
2. Despite this, NFT holders can still call `transferFrom()` or `safeTransferFrom()` to transfer their NFTs as usual, because these functions do not check the paused state.

### Impact

This behavior is inconsistent with the protocol’s intended design. Even when the `AlignerzNFT` contract is paused, NFT transfers remain possible.

### PoC

_No response_

### Mitigation

_No response_
  