# [000525] Stale allocation data in `allocationOf` mapping leads to incorrect information for external integrations and monitoring systems
  
  ### Summary


The `AlignerzVesting` contract maintains two separate storage locations for NFT allocations: one in the project-specific mappings (`biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]`) and another in the global `allocationOf[nftId]` mapping. However, when users claim tokens via `AlignerzVesting::claimTokens`, only the project-specific allocation is updated while `allocationOf` remains unchanged, causing permanent desynchronization.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L941
In `AlignerzVesting::claimTokens`, the function updates the project-specific allocation storage but never synchronizes the changes back to the `allocationOf` mapping:
```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    // ... claiming logic updates allocation.claimedSeconds[i], allocation.claimedFlows[i], allocation.isClaimed ...
    // allocationOf[nftId] is NEVER updated
}
```
The initial population of `allocationOf` occurs during NFT minting in functions like `AlignerzVesting::claimNFT` and `AlignerzVesting::_claimRewardTVS`:
```solidity
allocationOf[nftId] = biddingProject.allocations[nftId];
```

This creates a snapshot that becomes stale after any subsequent claims.

### Internal Pre-conditions

1. An NFT must have been minted with an allocation (via `AlignerzVesting::claimNFT` or `AlignerzVesting::_claimRewardTVS`)
2. The `allocationOf[nftId]` mapping must have been populated during NFT creation


### External Pre-conditions

1. User must call `AlignerzVesting::claimTokens` to claim vested tokens


### Attack Path

1. User claims NFT from a bidding project via `AlignerzVesting::claimNFT(0, 1, 1000, proof)`, receiving NFT ID 5
2. Contract sets `allocationOf[5]` with `claimedSeconds[0] = 0` and `claimedFlows[0] = false`
3. Time passes and tokens vest
4. User calls `AlignerzVesting::claimTokens(0, 5)` to claim 500 tokens
5. Contract updates `biddingProjects[0].allocations[5].claimedSeconds[0]` to reflect the claim
6. External monitoring system reads `allocationOf[5].claimedSeconds[0]` and still sees 0

### Impact

allocationOf will receive outdated information showing zero claimed seconds and incorrect vesting progress.



### PoC

No response

### Mitigation

Either remove the redundant allocationOf mapping and provide getter functions that read from project-specific storage, or ensure it stays synchronized:
  