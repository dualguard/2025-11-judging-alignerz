# [000585] Operator Delegation of Approval Rights Undermines NFT Owner Control Over Token Approvals
  
  
&nbsp;

## Summary

The ability for operators to delegate their approval rights via the `approve` function will cause a loss of control over token approvals for NFT owners as an operator will transfer approval rights to a third-party address before the owner revokes operator approval, leaving delegated approvals active even after the operator approval is removed.

&nbsp;

## Root Cause

In [`ERC721A.sol:257-266`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L257-L266), the `approve` function allows both token owners and operators (via `isApprovedForAll`) to set token-level approvals. When an operator calls `approve`, they can delegate approval rights to any address, including ones they control. In `ERC721A.sol:280-285`, the `setApprovalForAll` function only updates the `_operatorApprovals` mapping and does not clear any existing token-level approvals stored in `_tokenApprovals` that the operator may have set. 
```js
    function _approve(address to, uint256 tokenId, address owner) private {
        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }
```

&nbsp;

## Internal Pre-conditions

1. Token owner needs to call `setApprovalForAll(operator, true)` to grant operator approval to an address
2. Operator needs to call `approve(delegateAddress, tokenId)` to delegate approval rights to another address (potentially one they control) before the owner revokes operator approval
3. Token owner needs to call `setApprovalForAll(operator, false)` to revoke operator approval, which does not clear existing token-level approvals

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. Token owner calls `setApprovalForAll(operator, true)` to grant an operator permission to manage their tokens
2. Operator calls `approve(attackerAddress, tokenId)` to delegate approval rights to an address they control, setting `_tokenApprovals[tokenId] = attackerAddress`
3. Token owner calls `setApprovalForAll(operator, false)` to revoke operator approval, which only updates `_operatorApprovals[owner][operator] = false`
4. The `_tokenApprovals[tokenId]` mapping still contains `attackerAddress`, so `attackerAddress` retains approval to transfer the token
5. `attackerAddress` can now call `transferFrom` or `safeTransferFrom` to transfer the token, even though the operator approval has been revoked

&nbsp;

## Impact

NFT owners suffer a loss of control over their token approvals. Even after revoking operator approval, delegated approvals set by the operator remain active, allowing unauthorized third-party addresses to transfer tokens. The owner may be unaware of these delegated approvals and remain exposed to unexpected transfers. The attacker (or operator-controlled address) gains the ability to transfer tokens after the operator approval is revoked, undermining the owner's intent to remove delegated access.

&nbsp;

## Proof of Concept

N/A

&nbsp;

## Mitigation

1. Restrict operators from delegating approval rights: Modify the `approve` function in `ERC721A.sol:257-266` to only allow the token owner to call it, preventing operators from setting token-level approvals.
2. Clear token approvals when operator approval is revoked: Modify `setApprovalForAll` to clear all token-level approvals for tokens owned by the caller when revoking operator approval.

  