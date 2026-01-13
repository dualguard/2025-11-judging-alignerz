# [001080] Pause Does Not Stop AlignerzNFT Transfers, Leaving Protocol Without Emergency Freeze Control
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80

The above line states that when `AlignerzNFT::pause` is called, minting and transfers should both be paused.

However, the `whenNotPaused` modifier is only applied to mint and burn.
It is not applied to the transfer functions. This means that even **during an emergency, malicious users can still transfer** or sell their NFTs, despite the contract being “paused”.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L297

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L304

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L311

None of these functions include `whenNotPaused`.
As a result, the pause mechanism does not stop transfers, despite what the documentation claims.
During an emergency, attackers can freely move NFTs even when the protocol believes transfers are frozen.


### Internal Pre-conditions

_No response_

### External Pre-conditions

1. The protocol is in an emergency or under attack.
2. The owner calls `pause()` assuming that transfers are disabled.


### Attack Path

1. An attacker hacks the protcol
2. Owner calls pause() believing that minting and transfers are now disabled.
3. Attacker calls `transferFrom()` or `safeTransferFrom()` for any NFT and sell their NFTs.
4. Transfer succeeds because no pause checks exist on transfer functions.
5. Owner cannot prevent NFT movement during an exploit, liquidation, bidding issue, or governance emergency.

### Impact

The protocol loses administrative control over NFT transferability during emergencies.
No direct theft occurs, but the emergency stop becomes non-functional, removing a critical safety mechanism.

### PoC

It is acceptable to remove `whenNotPaused` from `mint` and `burn` because both operations internally call `_beforeTokenTransfers`, which can enforce pause checks.

```diff
contract AlignerzNFT{
    .
    .
    .
    .
    /**
     * @notice mints tokens based on parameters
     * @param to address of the user minting
     *
     */
-   function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
+   function mint(address to) external onlyMinter returns (uint256) {
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
-   function burn(uint256 tokenId) external whenNotPaused onlyMinter {
+   function burn(uint256 tokenId) external onlyMinter {
        require(_exists(tokenId), "ALIGNERZ: Token Id does not exist");
        _burn(tokenId);
    }

+   function _beforeTokenTransfers(address from, address to, uint256 startTokenId, uint256 quantity) internal override whenNotPaused {
+       super._beforeTokenTransfers(from,to,startTokenId,quantity);
+   }
}
```

### Mitigation

_No response_
  