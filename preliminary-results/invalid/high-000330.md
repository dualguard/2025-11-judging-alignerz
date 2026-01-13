# [000330] Front-Running updateProjectAllocations causes Double-Claim
  
  ### Summary


The [updateProjectAllocations](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L812) function allows the owner to update merkle roots after bidding has been finalized and users have started claiming. This creates a front-running opportunity where users can claim their allocations with old merkle proofs before the update executes, then claim again with new merkle proofs after the update. The vulnerability manifests in two primary scenarios: (1) users claiming NFT when their allocation amount or pool changes, and (2) users claiming both refunds and NFTs when their status changes from loser to winner. Both scenarios result in users receiving significantly more value than intended, causing direct financial loss to the project.

### Root Cause

The root cause stems from interconnected design flaws:

**1. Separate Claim Tracking Systems**

The contract uses two independent mappings to track claims:
- `claimedNFT[leaf]` for NFT claims
- `claimedRefund[leaf]` for refund claims

There is no cross-validation between these systems, allowing users to claim from both mappings for the same project.

**2. Leaf Hash Includes Mutable Parameters**

The leaf hash calculation includes parameters that can change during updates:

```javascript
bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
```

When `amount` or `poolId` changes via `updateProjectAllocations`, a completely different leaf hash is generated, bypassing the `Already_Claimed()` check even though the same user is claiming for the same project.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


### Scenario 1: Double NFT Claim via Amount/Pool Change

**Step 1: Initial State Setup**
- After `finalizeBids` is called, Alice is allocated to Pool 0 with 50,000 tokens
- The merkle root for Pool 0 contains: `hash(Alice, 50000, projectId, 0)`
- Alice has a valid merkle proof for this allocation

**Step 2: Owner Prepares Update**
- Owner realizes Alice's allocation should be 60,000 tokens (or moved to Pool 1)
- Owner generates new merkle tree with: `hash(Alice, 60000, projectId, 0)` or `hash(Alice, 50000, projectId, 1)`
- Owner prepares to call `updateProjectAllocations` with new merkle roots

**Step 3: Owner Submits Transaction**
- Owner calls `updateProjectAllocations(projectId, newRefundRoot, newMerkleRoots)`
- Transaction enters the public mempool with standard gas price
- Transaction is pending but not yet mined

**Step 4: Alice Detects and Front-Runs**
- Alice monitors mempool and detects the `updateProjectAllocations` transaction
- Alice sees her allocation will change (amount increase or pool change)
- Alice immediately submits `claimNFT(projectId, 0, 50000, oldProof)` with 2x gas price
- Alice's transaction is prioritized and mined first

**Step 5: Alice's First Claim Executes**
- `leaf1 = keccak256(abi.encodePacked(Alice, 50000, projectId, 0))`
- `require(!claimedNFT[leaf1])` passes (first claim)
- Merkle proof verified against old root: valid
- `claimedNFT[leaf1] = true`
- Alice receives NFT 1 with 50,000 tokens
- Transaction succeeds

**Step 6: Owner's Update Executes**
- Next block includes owner's `updateProjectAllocations` transaction
- Pool 0 merkle root updated to new value
- New root contains Alice with 60,000 tokens (or Pool 1 with 50,000 tokens)
- State change complete

**Step 7: Alice Claims Again**
- Alice calls `claimNFT(projectId, 0, 60000, newProof)` (or `claimNFT(projectId, 1, 50000, newProof)`)
- `leaf2 = keccak256(abi.encodePacked(Alice, 60000, projectId, 0))` - DIFFERENT LEAF
- `require(!claimedNFT[leaf2])` passes (different leaf hash)
- Merkle proof verified against new root: valid
- `claimedNFT[leaf2] = true`
- Alice receives NFT 2 with 60,000 tokens (or 50,000 from Pool 1)
- Transaction succeeds

**Step 8: Attack Complete**
- Alice now holds 2 NFTs with total of 110,000 tokens (or 100,000 if pool changed)
- Alice should only have 60,000 tokens (or 50,000)
- Project has lost 50,000 tokens to Alice's double-claim

---

### Scenario 2: Refund + NFT Double Claim via Status Change

**Step 1: Initial State Setup**
- After `finalizeBids`, Bob placed a 10,000 USDT bid but is marked as a loser
- Bob appears in the refund merkle tree: `hash(Bob, 10000, projectId, 2)`
- Bob has a valid merkle proof for his refund
- Bob does NOT appear in any pool's winner tree

**Step 2: Owner Prepares Update**
- Owner realizes Bob should actually be a winner with 20,000 tokens in Pool 2
- Owner generates new merkle trees:
  - Remove Bob from refund tree
  - Add Bob to Pool 2 winner tree: `hash(Bob, 20000, projectId, 2)`
- Owner prepares to call `updateProjectAllocations`

**Step 3: Owner Submits Transaction**
- Owner calls `updateProjectAllocations(projectId, newRefundRoot, newMerkleRoots)`
- Transaction enters mempool with standard gas price
- Transaction pending

**Step 4: Bob Detects and Front-Runs**
- Bob monitors mempool and detects the update transaction
- Bob sees he will become a winner with 20,000 tokens
- Bob immediately submits `claimRefund(projectId, 10000, oldRefundProof)` with 2x gas price
- Bob's transaction prioritized and mined first

**Step 5: Bob's Refund Claim Executes**
- `leaf1 = keccak256(abi.encodePacked(Bob, 10000, projectId, 0))`
- `require(!claimedRefund[leaf1])` passes (first refund claim)
- Merkle proof verified against old refund root: valid
- `claimedRefund[leaf1] = true`
- Bob receives 10,000 USDT refund
- `biddingProject.totalStablecoinBalance -= 10000`
- Transaction succeeds

**Step 6: Owner's Update Executes**
- Next block includes owner's transaction
- Refund root updated (Bob removed)
- Pool 2 merkle root updated (Bob added as winner)
- State changes complete

**Step 7: Bob Claims NFT**
- Bob calls `claimNFT(projectId, 2, 20000, newWinnerProof)`
- `leaf2 = keccak256(abi.encodePacked(Bob, 20000, projectId, 2))` - DIFFERENT LEAF
- `require(!claimedNFT[leaf2])` passes (different mapping and different leaf)
- Merkle proof verified against new Pool 2 root: valid
- `claimedNFT[leaf2] = true`
- Bob receives NFT with 20,000 tokens
- Transaction succeeds

**Step 8: Attack Complete**
- Bob received 10,000 USDT refund
- Bob received NFT with 20,000 tokens
- Bob should only have received 20,000 tokens (no refund)
- Project lost 10,000 USDT + gave Bob tokens he already "paid for" with the refund


### Impact


1. **Direct Financial Loss**: Immediate and quantifiable loss of both tokens and stablecoins
2. **Protocol Insolvency**: The project may not have sufficient tokens or stablecoins to fulfill all legitimate claims after the attack
3. **Market Impact**: Sudden influx of extra tokens from exploiters could crash the token price, harming all holders
4. **Affects Remaining users**:It affects all remaining users.


### PoC

_No response_

### Mitigation

 Implement Unified User-Level Claim Tracking which tracks user across the functions
  