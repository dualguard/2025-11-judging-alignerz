# [000471] Unnecessary External Call to ownerOf in `approve:;ERC721A.sol`
  
  ### Summary

The `approve` function in `ERC721A` contract unnecessarily performs an external call to `ownerOf` using `ERC721A(this).ownerOf(tokenId)` even though the function resides in the same contract. This forces the call to go through ABI encoding/decoding and an external call frame instead of using a cheaper, safer internal call. While this does not break correctness, it increases gas consumption, expands the external call surface, and decreases code clarity.

### Root Cause

Using `ERC721A(this).ownerOf(tokenId)` instead of an internal call (ownerOf(tokenId) or _ownerOf(tokenId)).
```solidity
function approve(address to, uint256 tokenId) public override {
    address owner = ERC721A(this).ownerOf(tokenId); // unnecessary external call
    ...
}
```

### Internal Pre-conditions

Not required.

### External Pre-conditions

Not required.

### Attack Path

Not an attack

### Impact

1. **Gas Inefficiency** – Performs a full external call instead of an internal dispatch.
2. **Unnecessary External Dispatch** – Adds extra call frame + ABI overhead.
3. **Slight Attack Surface Expansion** – External calls (even to self) introduce additional call context (new frame), making behavior less predictable and harder to audit. Can cause serious issues if overriden in a way which requires `msg.sender`, because external call changes `msg.sender`.
4. **Maintainability Issue** – The pattern looks like a cross-contract call, increasing reader confusion.

**Severity - LOW**.

### PoC

NA

### Mitigation

Just make an internal call to `ownerOf` function directly:-
```diff
function approve(address to, uint256 tokenId) public override {
+        address owner = ownerOf(tokenId);
        if (to == owner) revert ApprovalToCurrentOwner();

        if (_msgSender() != owner && !isApprovedForAll(owner, _msgSender())) {
            revert ApprovalCallerNotOwnerNorApproved();
        }

        _approve(to, tokenId, owner);
    }
```
  