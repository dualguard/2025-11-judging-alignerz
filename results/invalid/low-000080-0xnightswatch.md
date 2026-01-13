# [000080] Missing event emission in removeMinter prevents minter revocation tracking
  
  ### Summary

The missing event emission in `removeMinter` at line 143 will cause off-chain systems to miss critical access control revocations as the owner will remove minting privileges without any on-chain notification. [AlignerzNFT.sol:140-144](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L140-L144)

### Root Cause

In [AlignerzNFT.sol:143](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L143), the `removeMinter()` function modifies the `minters` mapping without emitting an event to notify off-chain systems of the access control change.

### Internal Pre-conditions

1. Admin call `removeMinter()`

### External Pre-conditions

None 

### Attack Path

1. Owner calls `emoveMinter()` to revoke minting privileges from an address
2. The `minters[newMinter]` mapping is set to false
3. No event is emitted to notify monitoring systems

### Impact

- Inability to audit minter role removals off-chain
- Security monitoring systems cannot track privilege changes
- Compliance and governance tracking becomes impossible
- Difficulty verifying that compromised addresses were properly revoked

### PoC

_No response_

### Mitigation

Add an event emission in `removeMinter():`

```diff
event MinterRemoved(address indexed minter);  
  
function removeMinter(address newMinter) external onlyOwner {  
    require(newMinter != address(0), "ALIGNERZ: Address cannot be 0");  
    require(minters[newMinter], "ALIGNERZ: Address not a minter");  
    minters[newMinter] = false;  
    emit MinterRemoved(newMinter);  
}
```
  