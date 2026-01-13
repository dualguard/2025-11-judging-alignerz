# [000204] Stale allocationOf Mapping Not Updated in claimTokens Causes Incorrect Dividend Calculations
  
  ### Summary

In `AlignerzVesting.sol:941-975`, the claimTokens function updates the canonical allocation struct `(biddingProjects[].allocations[] or rewardProjects[].allocations[])` but never syncs the public allocationOf mapping, which will cause unfair dividend over-distribution for users who have already claimed tokens as `A26ZDividendDistributor` will read stale `claimedSeconds=0` and `claimedFlows=false` values, treating fully-claimed positions as fully-unclaimed.

### Root Cause

In `AlignerzVesting.sol:941-975`, the claimTokens function modifies `allocation.claimedSeconds[i]` and `allocation.claimedFlows[i]` on the canonical storage location (`biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]`), but the public allocationOf[nftId] mapping which was set only once at NFT creation via struct copy—is never updated:

``` solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    // Gets pointer to CANONICAL storage
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    
    // ...
    for (uint256 i; i < nbOfFlows; i++) {
        // Updates CANONICAL only
        allocation.claimedSeconds[i] += claimableSeconds;
        if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
            allocation.claimedFlows[i] = true;
        }
    }
    // ...
    // allocationOf[nftId] is NEVER updated - remains stale snapshot from creation
}
```

### Internal Pre-conditions

1. User needs to have a TVS NFT created via claimNFT() or reward distribution functions
2. User needs to call claimTokens() at least once to create state divergence between canonical allocation and allocationOf snapshot
3. Owner needs to call setUpTheDividends() on A26ZDividendDistributor after token claims have occurred

### External Pre-conditions

None required.

### Attack Path

1. Alice and Bob both receive TVS NFTs with 1000e18 tokens each, vesting over 12 months
2. At creation, `allocationOf[aliceNftId].claimedSeconds[0] = 0` and `allocationOf[bobNftId].claimedSeconds[0] = 0`
3. Alice calls `claimTokens()` monthly for 10 months, claiming ~833e18 tokens
4. Canonical `biddingProjects[projectId].allocations[aliceNftId].claimedSeconds[0]` is now ~26,280,000 (10 months in seconds)
5. But `allocationOf[aliceNftId].claimedSeconds[0]` remains 0 (never updated)
6. Bob never claims any tokens (true diamond hands behavior)
7. Owner deposits 10,000 USDC to A26ZDividendDistributor and calls `setUpTheDividends()`
8. `getUnclaimedAmounts(aliceNftId)` reads `vesting.allocationOf(aliceNftId).claimedSeconds[0]` → returns 0
9. Both Alice and Bob are calculated as having ~1000e18 unclaimed tokens
10. Alice receives 5,000 USDC (50%) despite only having ~167e18 tokens remaining
11. Bob receives 5,000 USDC (50%) despite having the full 1000e18 tokens unclaimed
12. Bob (the intended "diamond hands" beneficiary) loses ~4,100 USDC in deserved dividends

### Impact

The A26ZDividendDistributor distributes rewards inversely to the protocol's stated "Diamond Hands Rewarder" design. Users who claimed most of their vested tokens receive equal or greater dividend shares compared to users who held through the entire vesting period. Long-term holders (the intended beneficiaries) suffer losses proportional to how much other users have claimed potentially 50-90% of their deserved dividends depending on the distribution of claim behavior across TVS holders.

### PoC

_No response_

### Mitigation

1. Sync allocationOf at the end of claimTokens:
``` solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    
    // ... existing claim logic ...
    
    if (flowsClaimed == nbOfFlows) {
        nftContract.burn(nftId);
        allocation.isClaimed = true;
    }
    token.safeTransfer(msg.sender, claimableAmounts);
    
    // Sync allocationOf with canonical allocation
    allocationOf[nftId] = allocation;
    
    emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed);
}
```
2. Alternatively, deprecate allocationOf entirely and provide a getter function that reads directly from the canonical source:
``` solidity
function getAllocation(uint256 projectId, uint256 nftId) external view returns (Allocation memory) {
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    return isBiddingProject ? 
        biddingProjects[projectId].allocations[nftId] : 
        rewardProjects[projectId].allocations[nftId];
}

```
  