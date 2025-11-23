# [000962] Users can transfer TVS during a pause
  
  ### Summary

In the `AlignerzNFT` contract, users can transfer NFTs during a pause, even though this should be prohibited. The `mint()` and `burn()` functions are protected by the `whenNotPaused` modifier, but the transfer functions do not check the pause status.

### Root Cause

The pause and unpause functions have the following natSpec comment: 
```solidity 
 /// @notice pauses the contract (minting and transfers)
```

The `AlignerzNFT` contract inherits `ERC721A` and `Pausable`. Transfer functions are inherited from `ERC721A` and are not redefined in `AlignerzNFT` with pause verification.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A pause is activated.
2. The user transfers their token.

### Impact

Users will be able to bypass this restriction and transfer tokens during the pause. 

### PoC

```solidity 
  function test_pausable() public {
        vm.startPrank(owner);
        nft.mint(bidders[0]);
        nft.pause();
        vm.stopPrank();
        vm.startPrank(bidders[0]);
        nft.transferFrom(bidders[0], bidders[1], 1);
        vm.stopPrank();
    }
```

### Mitigation

_No response_
  