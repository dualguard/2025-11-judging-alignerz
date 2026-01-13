# [000391] approve() does not check token validity
  
  **Summary:** Unlike other functions the `approve()` function in `ERC721A` contract does not check whether tokenId exist or not
```solidity
function approve(address to, uint256 tokenId) public override {
        address owner = ERC721A(this).ownerOf(tokenId);
        if (to == owner) revert ApprovalToCurrentOwner();

        if (_msgSender() != owner && !isApprovedForAll(owner, _msgSender())) {
            revert ApprovalCallerNotOwnerNorApproved();
        }
        _approve(to, tokenId, owner);
    }

    /**
     * @dev See {IERC721-getApproved}.
     */
    function getApproved(uint256 tokenId) public view override returns (address) {
        if (!_exists(tokenId)) revert ApprovalQueryForNonexistentToken();

        return _tokenApprovals[tokenId];
    }
```
**Recommendation:**
Add the check that for token must exist
```solidity
  if (!_exists(tokenId)) revert ApprovalQueryForNonexistentToken()
```
  