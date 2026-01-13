# [000141] Deployer address remains privileged minter after ownership transfer, creating hidden backdoor
  
  ### Summary

The AlignerzNFT contract grants minter privileges to the deployer in its constructor but does not revoke these rights when ownership is transferred to the protocol owner. This creates a backdoor where the deployer can mint and burn NFTs after losing control.

### Root Cause

In the deployment scripts (e.g., `a26zAmoy.s.sol`), the sequence is: deploy contract, add vesting as minter, then call `nft.transferOwnership(newOwner)`. However, `transferOwnership()` does not affect the minters mapping, leaving the deployer's privileges intact.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L45-L51

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

.

### Impact

While the protocol owner now controls the contract, the deployer retains minting and burning capabilities. Although unlikely to be abused in practice, this violates the principle of least privilege and creates unnecessary risk if the deployer account is compromised.


### PoC

_No response_

### Mitigation

After transferring ownership in the deployment script, explicitly call `nft.removeMinter(deployerAddress)` to clean up privileges.
  