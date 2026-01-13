# [000853] Cross-Function Merkle Proof Replay Attack Between Claim Functions
  
  ## Summary

The `claimRefund()` and `claimNFT()` functions use identical leaf construction when `poolId = 0` and amounts match, enabling replay attacks where users can use proofs intended for one function to claim from another function they're not entitled to.

## Vulnerability Detail

Both claim functions construct Merkle tree leaves using identical parameters and hashing methods:

**Locations**: 
- [`AlignerzVesting.sol:844`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L844) - claimRefund
- [`AlignerzVesting.sol:871`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L871) - claimNFT

```solidity
// claimRefund() - Line 844
function claimRefund(uint256 amount, uint256 projectId, bytes32[] calldata merkleProof) external {
    uint256 poolId = 0; // Always 0 for refunds
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, refundAllocationRoot[projectId], leaf), "Invalid proof");
    // ... claim logic
}

// claimNFT() - Line 871  
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof) external {
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, allocationRoots[projectId][poolId], leaf), "Invalid proof");
    // ... claim logic
}
```

**Root Cause**: When both functions use `poolId = 0` and the same `amount`, they generate identical leaf hashes, making valid proofs interchangeable between functions.

### Attack Mechanism

1. **Setup Phase**: User has valid allocation in both refund (amount X) and NFT pool 0 (same amount X) for the same project
2. **Hash Collision**: Both functions generate identical leaf hashes due to same parameters
3. **Replay Exploitation**: User can use their valid NFT claim proof to claim a refund (or vice versa) they're not entitled to
4. **Double Claims**: User effectively claims from both pools using a single legitimate proof

**Example Scenario**:
```solidity
// User has 1000 tokens in refund allocation
claimRefund(1000, projectId=1, proof_A) 
// Generates: keccak256(abi.encodePacked(user, 1000, 1, 0))

// User also has 1000 tokens in NFT pool 0  
claimNFT(projectId=1, poolId=0, amount=1000, proof_B)
// Generates: keccak256(abi.encodePacked(user, 1000, 1, 0))

// Same hash! User can replay proof_A to claim NFT or proof_B to claim refund
```

## Impact

Cross-function authentication bypass enabling unauthorized claims

- **Unauthorized Double Claims**: Users claim both refunds and NFTs when only entitled to one
- **Fund Drainage**: Protocol loses additional tokens through invalid claims  
- **Allocation Bypass**: Users circumvent intended allocation restrictions
- **Proof Reuse**: Single valid proof becomes usable across multiple claim types

**Attack Requirements**:
- User must have legitimate allocation in both refund and NFT pool 0
- Amounts must match between allocations
- Same project must be involved

## Code Snippet

**Vulnerable leaf construction in both functions**:
```solidity
function claimRefund(uint256 amount, uint256 projectId, bytes32[] calldata merkleProof) external {
    // ... validation ...
    uint256 poolId = 0; // Fixed value for refunds
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, refundAllocationRoot[projectId], leaf), "Invalid proof");
    // ... claim logic
}

function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof) external {
    // ... validation ...
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, allocationRoots[projectId][poolId], leaf), "Invalid proof");
    // ... claim logic
}
```

**Hash collision example**:
```solidity
// Both generate identical hashes when poolId = 0:
bytes32 refundLeaf = keccak256(abi.encodePacked(user, 1000, 1, 0));
bytes32 nftLeaf = keccak256(abi.encodePacked(user, 1000, 1, 0));
// refundLeaf == nftLeaf when parameters match
```


## Recommendation

Add function type differentiation to leaf construction.

```solidity
// For refunds
function claimRefund(uint256 amount, uint256 projectId, bytes32[] calldata merkleProof) external {
    bytes32 leaf = keccak256(abi.encode("REFUND", msg.sender, amount, projectId));
    require(MerkleProof.verify(merkleProof, refundAllocationRoot[projectId], leaf), "Invalid proof");
    // ... rest of function
}

// For NFT claims
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof) external {
    bytes32 leaf = keccak256(abi.encode("NFT_CLAIM", msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, allocationRoots[projectId][poolId], leaf), "Invalid proof");
    // ... rest of function
}
```

  