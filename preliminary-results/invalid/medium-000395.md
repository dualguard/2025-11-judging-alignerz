# [000395] Insecure Merkle Root Setting Allows Permanent Loss of User Claims
  
  **Summary:**
The `updateProjectAllocations()` function updates all allocation Merkle roots and the refund root after bidding closes.
However, there is zero validation that the supplied roots are correct, consistent, or correspond to the actual bids.

As a result:
The owner can accidentally upload wrong Merkle roots.
Once wrong roots are stored, the contract has no restore mechanism, leading to permanent loss of withdrawals, allocations, NFTs, and refunds for users. Both allocation tree and refund tree are affected. This creates a single-point-of-failure, breaking the entire trust model of the protocol.

```solidity
 function updateProjectAllocations(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(biddingProject.closed, Project_Still_Open());
        require(merkleRoots.length == biddingProject.poolCount, Invalid_Merkle_Roots_Length());

        // Set merkle root for each pool
        for (uint256 poolId = 0; poolId < biddingProject.poolCount; poolId++) {
            biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
            emit PoolAllocationSet(projectId, poolId, merkleRoots[poolId]);
        }

        biddingProject.refundRoot = refundRoot;
        emit AllPoolAllocationsSet(projectId);
    }
```

**Impact:**
If incorrect roots are set:
- Users cannot claim NFTs (allocation Merkle root mismatch)
- Users cannot claim refunds (refundRoot mismatch)
- Claimed flags cannot be reset (permanent lock)

**Root cause**
No validation of Merkle roots. The following code simply overwrites roots without any checks:
```solidity
biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
biddingProject.refundRoot = refundRoot;
```
  