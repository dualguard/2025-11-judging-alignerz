# [000833] The transfer of NFT is possible when `AlignerzNFT` contract is paused
  
  ### Summary

The users can transfer tokens when the contract is paused.

### Root Cause

The `AlignerzNFT` contract has a pausable mechanism and doesn't allow the `burn` and `mint` functions to be execute when the contract is paused:

```solidity

function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
    require(to != address(0), "ALIGNERZ: Address cannot be 0");

    uint256 startToken = _currentIndex;
    _safeMint(to, 1);

    emit Minted(to, startToken);
    return startToken;
}

/**
 * @notice a burn function for burning specific tokenId
 * @param tokenId Id of the Token
 *
 */
function burn(uint256 tokenId) external whenNotPaused onlyMinter {
    require(_exists(tokenId), "ALIGNERZ: Token Id does not exist");
    _burn(tokenId);
}

```

The problem is that the contract allows the `transfer` function to be executed when the contract is paused. The contract inherits the `ERC721A` contract that defines the transfer functionality. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

There is no attack path. The transfer functionality can be used when the contract is paused.

### Impact

When the contract is paused, users can still perform state-changing operations that might be undesirable during a pause period. It could lead to unexpected state changes during contract emergency situations.


### PoC

_No response_

### Mitigation


Create transfer function in `AlignerzNFT` contract that calls the transfer function from `ERC721A` contract and add `whenNotPaused` modifier.
  