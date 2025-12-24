# [000749] Hard `claimDeadline` in `claimNFT` lets off-chain outages strand users’ TVS allocations
  
  
### Summary

The NFT claim flow uses a strict on-chain deadline, so if users cannot obtain their `poolId` or Merkle proof in time (for example due to a frontend or backend outage, DDOS), their TVS allocation becomes permanently unclaimable even though the funds remain in the protocol.

### Root Cause

In `claimNFT`, the contract enforces that claims must happen strictly before `claimDeadline` without any admin recovery path or grace mechanism:

```solidity
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
    external
    returns (uint256)
{
    BiddingProject storage biddingProject = biddingProjects[projectId];
>>  require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
    ...
}
```

If, near the end of the claim window, users still do not know their exact `(poolId, amount, proof)` (because this data is only exposed off-chain) and the UI/API goes down or is attacked, they have no way to claim on-chain before `claimDeadline`, and once the deadline has passed, they are blocked forever by this `require`.

### Impact

There is no external attacker path inside the contract itself, but a website/API DoS or operational mistake can permanently lock users’ claimable TVS/NFTs because the smart contract strictly enforces `block.timestamp < claimDeadline` and does not allow the trusted owner to extend or override the deadline for users who were unable to claim in time.

### Mitigation

Soften the liveness constraint,  allow the owner to extend `claimDeadline` for a project so they can react to outages and give users extra time.  

  