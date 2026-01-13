# [000261] Pool Capacity and Allocation Ordering Not Enforced in claimNFT()
  
  ### Summary

The claimNFT() function allows users to claim NFTs corresponding to accepted bids using a Merkle proof for verification:

```
bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
require(!claimedNFT[leaf], Already_Claimed());
require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());
```

While the Merkle proof enforces that a user is eligible for a given pool (poolId) and bid amount, the contract does not enforce pool capacities or claim order on-chain.

Pool assignment is determined entirely off-chain via the Merkle tree:

```
function updateProjectAllocations(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots) external onlyOwner {
    for (uint256 poolId = 0; poolId < biddingProject.poolCount; poolId++) {
        biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
    }
}
```


The contract stores only the root of each pool’s Merkle tree, not the actual pool membership or allocation counts.

As a result, there is no on-chain tracking of claimed allocations per pool.

### Root Cause

The verify and processProof functions ensure that only valid leaves can be claimed:

```
function verify(bytes32[] memory proof, bytes32 root, bytes32 leaf, function(bytes32, bytes32) view returns (bytes32) hasher) internal view returns (bool) {
    return processProof(proof, leaf, hasher) == root;
}

function processProof(bytes32[] memory proof, bytes32 leaf, function(bytes32, bytes32) view returns (bytes32) hasher) internal view returns (bytes32) {
    bytes32 computedHash = leaf;
    for (uint256 i = 0; i < proof.length; i++) {
        computedHash = hasher(computedHash, proof[i]);
    }
    return computedHash;
}
```

The proof ensures that the (msg.sender, amount, projectId, poolId) tuple exists in the off-chain Merkle tree. Any deviation (wrong poolId or amount) causes the claim to fail.


Even with Merkle verification, Pool capacity is not enforced on-chain. There is no counter or limit per pool to prevent over-allocation.

If the off-chain tree contains more entries than the intended pool size, the pool can be overfilled.

- Claim order is not enforced

- Users with valid leaves can claim NFTs in any order.

- Early bidders have no on-chain guarantee of priority; “first bid, first served” is violated.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Here is a clean and technically correct **Exploit Scenario** section:

---

## **Exploit Scenario**

Consider a project where **Pool 0** is intended for the earliest bidders and is capped at **500 NFTs**. The off-chain bidding engine collects all bids and is responsible for assigning users to pools before generating the Merkle tree.

1. **Off-chain assigns more than 500 users to Pool 0**

   * Due to incorrect or missing off-chain validation, **650 users** end up included in Pool 0’s Merkle tree.
   * All 650 leaves contain:

     ```
     keccak256(abi.encodePacked(user, amount, projectId, poolId))
     ```

2. **All 650 users receive valid Merkle proofs**

   * Even though only 500 should qualify, every user can prove membership because the on-chain contract trusts the root unconditionally.

3. **Claiming opens on-chain**

   * Users call:

     ```solidity
     claimNFT(projectId, amount, merkleProof)
     ```

     with the leaf:

     ```
     keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId))
     ```

     and the on-chain logic only checks:

     * the Merkle proof is valid
     * the leaf has not been claimed before

   * The contract **does not check pool caps**.

4. **Late bidders with valid proofs claim before early bidders**

   * Example sequence:

     * Attacker (a late bidder) claims early.
     * They occupy one of the 500 available NFT slots.
     * 150 late bidders do the same.

5. **Early bidders are pushed out**

   * When genuine early bidders attempt to claim, they find:

     * the pool is already exhausted, or
     * the downstream vesting/allocation logic fails because more users exist than expected.

6. **Result**

   * Late bidders gain allocations that were never intended for them.
   * Early bidders lose priority or receive no NFTs at all.
   * Project economics relying on capped pool sizes collapse.
   * Protocol behavior diverges from off-chain assumptions, breaking fairness guarantees.

---


### Impact

Pool Over-allocation

Without on-chain enforcement of pool capacities, multiple users can claim NFTs from the same pool beyond its intended limit.

This can lead to over-distribution of NFTs or vesting allocations, breaking the economic assumptions of the bidding system.

Violation of Claim Order / Fairness

Early bidders may lose priority because claim order is not enforced.

“First bid, first served” or time-based allocation guarantees are effectively bypassed, undermining fairness.

### PoC

_No response_

### Mitigation

Track pool allocations on-chain

Maintain a counter per pool and enforce maximum capacity.

Enforce claim order (optional, if fairness is critical)

Consider assigning sequential claim indices or restricting claims based on bid timestamp to enforce “first bid, first served”.
  