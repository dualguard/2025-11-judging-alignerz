# [000684] [H-1] Merkle leaf collision blocks legitimate allocations
  
  ### Summary

The Merkle leaf does not include a per-allocation nonce/index, so identical `(user, amount, projectId, poolId)` leaves collide; this causes a permanent denial of valid NFT claims for users when the backend emits duplicate-sized allocations in the same pool, as the first claim sets `claimedNFT[leaf]` and all subsequent claims revert (`Already_Claimed`).

### Root Cause

In `AlignerzVesting.sol:870-879` the allocation leaf is `keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId))` and `claimedNFT[leaf]` is used to gate claims, with no unique per-allocation nonce/index to clear up multiple identical allocations.

### Internal Pre-conditions

  1. Backend includes at least two identical leaves for the same user and pool with the same `amount` in the Merkle tree.
  2. The user has an existing bid in the project and the claim window is still open.

### External Pre-conditions

None. 

### Attack Path

  1. User (or any claimant) calls `claimNFT(projectId, poolId, amount, proof)` for the first identical leaf; Merkle verifies, `claimedNFT[leaf]` flips to true, NFT mints.
  2. User calls `claimNFT` again with the second identical leaf; Merkle verifies but `claimedNFT[leaf]` is already true, so the call reverts with `Already_Claimed`, permanently blocking the allocation.

### Impact

The affected user cannot claim the additional allocation(s); those tokens remain locked/unclaimable. Value at risk is the full size of each colliding allocation.

### PoC

I think it's optional in this scenario? 

### Mitigation

Add a per-allocation nonce/index to the leaf (and to the `claimedNFT` key), e.g., `keccak256(abi.encodePacked(user, amount, projectId, poolId, nonce))`, and propagate the nonce through off-chain tree generation and claim parameters. Alternatively, enforce uniqueness off-chain and reject duplicate leaves at generation time.
  