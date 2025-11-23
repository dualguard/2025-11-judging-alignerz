# [000750] Updating allocation Merkle roots after closing lets users double claim refunds/NFTs under old and new trees
  
  ### Summary

The ability to change refund and allocation Merkle roots after bids are finalized will cause double-claiming of refunds and NFTs for users as a bidder can claim once under the old tree and again under the new tree because claims are keyed by leaf, not by `(user, projectId, poolId)`.

### Root Cause

In `updateProjectAllocations` the owner can overwrite all pool Merkle roots and the refund root after the project is closed, and `claimedRefund`/`claimedNFT` track claimed status per leaf (hash of `(user, amount, projectId, poolId)`), not per `(user, projectId, poolId)`, so a user included in both the old and new trees has two different leaves and can claim twice.

```solidity
function updateProjectAllocations(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots)
    external
    onlyOwner
{
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
>>  require(biddingProject.closed, Project_Still_Open());
    require(merkleRoots.length == biddingProject.poolCount, Invalid_Merkle_Roots_Length());

    // Set merkle root for each pool
>>  for (uint256 poolId = 0; poolId < biddingProject.poolCount; poolId++) {
>>      biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
    }

>>  biddingProject.refundRoot = refundRoot;
    emit AllPoolAllocationsSet(projectId);
}
```

```solidity
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());

    uint256 poolId = 0;
>>  bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
>>  require(!claimedRefund[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
    claimedRefund[leaf] = true;
    ...
}
```

```solidity
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
    external
    returns (uint256)
{
    ...
>>  bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
>>  require(!claimedNFT[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());
    claimedNFT[leaf] = true;
    ...
}
```

When the owner updates roots, off-chain they will typically build the new tree assuming the current on-chain state (who has claimed, who has not). But there is no on-chain coordination: if a user U front-runs `updateProjectAllocations` and claims against `leaf_old` from the old tree, and the new tree still includes U with a possibly different `(amount, poolId)` producing `leaf_new`, U can later claim again under `leaf_new` because `claimedRefund[leaf_new]` / `claimedNFT[leaf_new]` is still `false`.

### Internal Pre-conditions

1. A bidding project has been closed via `finalizeBids`, so `biddingProject.closed == true` and initial `refundRoot` and `vestingPools[poolId].merkleRoot` have been set.  
2. At least one user `U` is present in both the old and intended new Merkle trees (same or different amount/poolId), and `U` has not yet claimed at the time the off-chain snapshot for the new tree is taken.

### External Pre-conditions

1. Users can see the owner’s `updateProjectAllocations` transaction in the mempool and send their own transactions in the same block (normal public mempool / miner ordering environment).  

### Attack Path

1. The owner closes the bidding project with `finalizeBids`, sets initial Merkle roots, and users start claiming refunds or NFTs with `claimRefund` / `claimNFT`.  
2. Later, the owner decides to fix allocations and off-chain rebuilds new Merkle trees that still include user `U` (because at snapshot time `U` has not claimed yet), and prepares a transaction for `updateProjectAllocations(projectId, newRefundRoot, newMerkleRoots)`.  
3. `U` observes this transaction in the mempool and front-runs it by calling `claimRefund` or `claimNFT` using the old root and proving `leaf_old = keccak256(abi.encodePacked(U, amount_old, projectId, poolId_old))`.  
4. The old claim succeeds and sets `claimedRefund[leaf_old] = true` or `claimedNFT[leaf_old] = true`.  
5. The owner’s `updateProjectAllocations` tx is then mined, installing `refundRoot = refundRoot_new` and `vestingPools[poolId].merkleRoot = merkleRoot_new[poolId]`, where `U` is included as `leaf_new = keccak256(abi.encodePacked(U, amount_new, projectId, poolId_new))`.  
6. Later, `U` calls `claimRefund` / `claimNFT` again with a proof for `leaf_new` under the new tree. Because `claimedRefund` / `claimedNFT` is checked per leaf, not per `(user, projectId, poolId)`, `claimedRefund[leaf_new]` and `claimedNFT[leaf_new]` are still false, so the second claim also succeeds.  
7. `U` has now double-claimed (once under `leaf_old`, once under `leaf_new`), draining extra stablecoins or TVS from the project.

### Impact

The bidding project’s funds for refunds and/or TVS allocations can be drained beyond the intended allocation set, as any user included in both old and new trees can claim twice if they front-run or simply claim once before and once after an `updateProjectAllocations` call. This can happen organically in a block tho. In the worst case, if the owner regularly uses `updateProjectAllocations` and users coordinate to exploit this, total payouts can approach “intended payouts × number of root updates,” leading to significant overpayment from the project’s balances.

### PoC



### Mitigation


Change `claimedRefund` and `claimedNFT` to be keyed by `(projectId, poolId, user)` (and possibly a type flag) instead of by `leaf` so that once a user has claimed for a given `(projectId, poolId)`, changing the Merkle tree cannot reset their claimed status:  
   ```solidity
   mapping(uint256 => mapping(uint256 => mapping(address => bool))) public refundClaimed;
   mapping(uint256 => mapping(uint256 => mapping(address => bool))) public nftClaimed;
   ```  
   and update checks to `require(!refundClaimed[projectId][poolId][msg.sender], Already_Claimed());`.


  