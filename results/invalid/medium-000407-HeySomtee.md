# [000407] ERC721A `_transfer` and `_burn` Do Not Clear Operator Approvals
  
  ### Summary

The `_transfer` and `_burn` functions in `ERC721A.sol` clear per-token approvals (`_tokenApprovals`) but do not revoke operator-level approvals set via `setApprovalForAll` (`_operatorApprovals`). This allows operators who were approved by a previous owner to retain transfer privileges after the token changes hands or is burned, violating the expected ERC721 security model where operator approvals are scoped to the approver's ownership period.

### Root Cause

Both `_transfer` and `_burn` call `_approve(address(0), tokenId, from)` to clear token-specific approvals, but the `_operatorApprovals` mapping (which stores `setApprovalForAll` permissions) is never modified. The ERC721A implementation does not include logic to automatically revoke operator approvals when ownership changes, relying on the assumption that operators are trusted and approvals are managed manually.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Relevant code snippets:
> In _transfer()
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L419-L420
> In _burn()
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L482-L483

```solidity
function _transfer(address from, address to, uint256 tokenId) private {
    // ... ownership update logic ...
    
    // Clear approvals from the previous owner
    _approve(address(0), tokenId, from);
    // @audit does not clear _operatorApprovals, i.e. isApprovedForAll
    
    // ... emit Transfer ...
}

function _burn(uint256 tokenId, bool approvalCheck) internal virtual {
    // ... burn logic ...
    
    // Clear approvals from the previous owner
    delete _tokenApprovals[tokenId];
    // @audit does not clear _operatorApprovals, i.e. isApprovedForAll
    
    // ... emit Transfer to zero address ...
}

function isApprovedForAll(address owner, address operator) public view virtual override returns (bool) {
    return _operatorApprovals[owner][operator];
}
```

1. Alice owns `tokenId = 5` and calls `setApprovalForAll(Bob, true)`, granting Bob operator status.
2. Alice transfers `tokenId = 5` to Carol via `transferFrom(Alice, Carol, 5)`.
3. `_transfer` clears `_tokenApprovals[5]` but leaves `_operatorApprovals[Alice][Bob] = true` unchanged.
4. Bob can still call functions checking `isApprovedForAll(Alice, Bob)` which returns `true` such as `transfer functions` and `approve()`, but Bob should no longer have any authority over tokens now owned by Carol.

### Impact

- **Stale operator privileges**: After burning tokens, operators remain approved and can transfer or approve arbitrary addresses on that particular token


### PoC

_No response_

### Mitigation

_No response_
  