# [000444] The NFT contract implements pausable but does not enforce it for transfers
  
  ### Summary

The contract `AlignerzNFT` handles a pausable mechanism to prevent the use of core functions in case of an emergency. However, it does not enforce the transfer mechanism to fail when the contract is paused, allowing for unauthorized NFT transfers. 

### Root Cause

`AlignerzNFT` does not implement pausable for NFT transfer.

### Attack Path

1. The owner pauses the protocol to freeze operations.
2. Users can still transfer tokens.

### Impact

Users bypass the pausable implementation.

### Mitigation

Override the transfer functions to implement `whenNotPaused` to them.
  