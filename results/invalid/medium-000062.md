# [000062] Broken Commit-Reveal: endTimeHash Is Never Verified in finalizeBids()
  
  ## Summary

The [`finalizeBids()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783) function lacks any mechanism to verify the `endTimeHash` commitment. The whitepaper promises a reveal phase where "we will reveal the exact input used to generate the hash, ensuring fairness," but the function signature doesn't accept a reveal parameter and never checks the stored hash. Instead, it just sets `endTime = block.timestamp`, allowing the admin to close auctions at any time without proving they honored their commitment.

## Vulnerability Details

The whitepaper describes a commit-reveal scheme where the admin commits to a bidding duration by hashing an unconventional string like `"OnEhOUR :+ tHirtY minutEs +-@ 20 fIve sECoNDS"`. The promise is explicit:

> "After the bidding ends, we will reveal the exact input used to generate the hash, ensuring fairness."

This is a standard cryptographic commitment scheme - the admin proves they didn't manipulate timing by revealing the preimage that matches the committed hash.

The function doesn't accept any reveal input, doesn't verify anything against `endTimeHash`. The stored hash is never read or checked - it's completely dead code.

## Impact

The whitepaper's "ensuring fairness" claim is false. Without verification, there's no cryptographic proof that the admin honored their commitment. They could commit to "7 days" but close after 3 days, and users have no way to prove dishonesty on-chain.

Even though the "Admin is fully trusted," the whitepaper explicitly describes a trustless cryptographic mechanism. Users reading the documentation expect mathematical guarantees, not trust-based discretion. This is a fundamental mismatch between promised and delivered security.

Consider how the protocol handles merkle proofs correctly:

```solidity
require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
```

The backend generates proofs off-chain, but the contract verifies them on-chain. The same pattern should apply here - if the admin reveals off-chain (e.g., on a website), users still can't verify honesty without on-chain checks.

## Severity

### Medium

- Broken functionality - The fairness mechanism described in whitepaper doesn't exist
- Misleading documentation - Users expect cryptographic guarantees but get trust-based system
- Missing security feature - The "ensuring fairness" promise is objectively false without verification


## Recommended Mitigation

Implement the reveal mechanism as described in the whitepaper:

```solidity
function finalizeBids(
    uint256 projectId,
    string calldata revealedInput,  // ← Add reveal parameter
    bytes32 refundRoot,
    bytes32[] calldata merkleRoots,
    uint256 claimWindow
) external onlyOwner {
    BiddingProject storage biddingProject = biddingProjects[projectId];

    // Verify the reveal matches the commitment
    bytes32 calculatedHash = keccak256(abi.encodePacked(revealedInput));
    require(calculatedHash == biddingProject.endTimeHash, "Invalid reveal");

    // Parse the revealed input to get the duration
    uint256 duration = parseDuration(revealedInput);
    uint256 committedEndTime = biddingProject.startTime + duration;

    // Ensure we're past the committed end time
    require(block.timestamp >= committedEndTime, "Too early");

    // Set end time based on commitment, not current time
    biddingProject.endTime = committedEndTime;
    biddingProject.closed = true;

    emit BiddingWindowRevealed(projectId, revealedInput, committedEndTime);
    // ... rest of logic ...
}

// Helper to parse unconventional time formats
function parseDuration(string memory input) internal pure returns (uint256) {
    // Parse formats like "OnEhOUR :+ tHirtY minutEs +-@ 20 fIve sECoNDS"
    // Return duration in seconds
}
```

  