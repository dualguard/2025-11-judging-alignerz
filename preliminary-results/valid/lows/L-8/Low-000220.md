# [000220] Pausing Does Not Disable Transfers in AlignerzNFT, Breaking Expected Security Guarantees
  
  ### Summary

The [`pause()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80-L83) function in `AlignerzNFT` is documented as disabling minting and transfers, yet the current implementation pauses only minting and burning. Transfers (transferFrom / safeTransferFrom) remain fully functional during pause, contradicting the intended and stated behavior.
This leads to a mismatch between expected guarantees during emergency pause and actual access control, potentially enabling undesirable vesting-entitlement NFT movements during critical situations.

### Root Cause

In `AlignerzNFT.sol`, the `pause()` and `unpause()` functions only apply `whenNotPaused` to minting/burning, but the contract does not override `ERC721A` transfer functions, so transfers never check whether the contract is paused/unpaused.
As a result, transfers remain active during emergency pause.

* mint()/burn() - pausable
```js
    function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
        ...
    }

    function burn(uint256 tokenId) external whenNotPaused onlyMinter {
        ...
    }
```

* transferFrom()/safeTransferFrom() - not pausable
```js
    function transferFrom(address from, address to, uint256 tokenId) public virtual override {
        _transfer(from, to, tokenId);
    }

    function safeTransferFrom(address from, address to, uint256 tokenId) public virtual override {
        safeTransferFrom(from, to, tokenId, "");
    }

    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory _data) public virtual override {
        _transfer(from, to, tokenId);
        if (isContract(to) && !_checkContractOnERC721Received(from, to, tokenId, _data)) {
            revert TransferToNonERC721ReceiverImplementer();
        }
    }
```

### Internal Pre-conditions

1. Owner needs to call `pause()` to pause the AlignerzNFT contract.
2. The paused state variable must be set to true.

### External Pre-conditions

None.

### Attack Path

1. Admin pauses the AlignerzNFT contract during emergency expecting mint/burn/transfer to be disabled.
2. A user (or attacker) simply calls `transferFrom()` or `safeTransferFrom()` on a paused contract.
3. Transfer succeeds because ERC721A never checks `whenNotPaused`.
4. Vesting entitlement NFTs are moved during an emergency pause, bypassing intended freeze controls.

### Impact

The protocol cannot guarantee that vesting-entitlement NFTs remain frozen during emergency pause and users can freely move, sell, or transfer their vesting positions during a pause, defeating the purpose of the pause mechanism. This breaks a core security guarantee.

### PoC

None.

### Mitigation

One solution is to override the `_beforeTokenTransfers()` in `AlignerzNFT` and add `whenNotPaused`:
```js
function _beforeTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal override whenNotPaused {
    super._beforeTokenTransfers(from, to, startTokenId, quantity);
}
```
  