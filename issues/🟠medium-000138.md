# [000138] Pausing does not enforce ERC721 transfer restrictions despite documentation
  
  ### Summary

The `AlignerzNFT` contract exposes `pause()` and `unpause()` functions, which are documented as pausing "minting and transfers." However, the paused state is only enforced on `mint()` and `burn()`, not on transfers. All transfer functions inherited from `ERC721A` remain fully usable while the contract is paused. This is a medium‑severity issue because it breaks an expected administrative control function. 

### Root Cause

`AlignerzNFT` inherits `Pausable` and exposes `pause()` / `unpause()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80-L88

The `whenNotPaused` modifier is applied only to `mint()` and `burn()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L106-L124

All transfer logic is implemented in `ERC721A` via `transferFrom()` and `safeTransferFrom()`, which call `_transfer()` and the hooks `_beforeTokenTransfers()` / `_afterTokenTransfers()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L297-L299


`AlignerzNFT` does not override `_beforeTokenTransfers()` or any transfer-related function to incorporate `whenNotPaused`. As a result, pausing the contract only affects minting and burning, while transfers remain unrestricted. This contradicts the comment that pause affects "minting and transfers" and undermines the expected ability to freeze secondary movement in emergencies.

### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

- Admin pauses the contract for any reason (incident response, upgrade)
- Pausing does not work as intended; NFTs can still be transferred

### Impact

This security oversight leads to a broken core functionality (pausing). The protocol can not use this functionality in emergencies. 

### PoC

_No response_

### Mitigation

One approach could be to override the ERC721A hooks in `AlignerzNFT` and apply `whenNotPaused`
  