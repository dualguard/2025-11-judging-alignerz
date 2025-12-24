# [000081] Missing event emission in addMinter prevents minter role tracking
  
  ### Summary

The missing event emission in `addMinter` at line 133 will cause off-chain systems to miss critical access control changes as the owner will grant minting privileges without any on-chain notification.
[AlignerzNFT.sol:130-134](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L130-L134)

### Root Cause

In A[lignerzNFT.sol:133](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L133), the `addMinter()` function modifies the minters mapping without emitting an event to notify off-chain systems of the access control change.

### Internal Pre-conditions

1. Admit call `addMinter()`

### External Pre-conditions

None

### Attack Path

1. Owner calls `addMinter()` to grant minting privileges to a new address
2. The `minters[newMinter]` mapping is set to `true`
3. No event is emitted to notify monitoring systems

### Impact

- Inability to audit minter role changes off-chain
- Security monitoring systems cannot track privilege escalation
- Compliance and governance tracking becomes impossible
- Difficulty investigating unauthorized minting incidents

### PoC

N/A

### Mitigation

Add an event emission in addMinter():

```diff
event MinterAdded(address indexed minter);  
  
function addMinter(address newMinter) external onlyOwner {  
    require(newMinter != address(0), "ALIGNERZ: Address cannot be 0");  
    require(!minters[newMinter], "ALIGNERZ: Address already a minter");  
    minters[newMinter] = true;  
    emit MinterAdded(newMinter);  
}
```
  