# [001004] smart account and wallets are prevented from minting
  
  ### Summary

The usage of _safeMint with the uage of the isContract check will cause a Denial of Service for Smart Contract Wallets as the protocol will revert transactions to contracts without onERC721Received.


### Root Cause

In [AlignerzNFT.sol:110](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L110)  the usage of _safeMint enforces that if the recipient is a contract, it must implement onERC721Received, but with smart account this break it and all smart account supported wallet will be dosed 

```solidity 
// AERC721.sol
function isContract(address addr) internal view returns (bool) {
    return addr.code.length > 0;
}

// AlignerzNFT.sol
function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
    require(to != address(0), "ALIGNERZ: Address cannot be 0");

    uint256 startToken = _currentIndex;
    _safeMint(to, 1); // <--- Enforces onERC721Received check

    emit Minted(to, startToken);
    return startToken;
}
```

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. User (e.g., a DAO admin) calls mint(address to) where to is their Gnosis Safe address or with a smart account.
The transaction reverts with TransferToNonERC721ReceiverImplementer.

### Impact

The Smart Contract Wallets and Smart Account  cannot mint NFTs

### PoC

_No response_

### Mitigation

1. modify the isContract to add smart account 
2. use the mint function 
  