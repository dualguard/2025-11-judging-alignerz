# [000574] Failure to Extend Claim Deadline After Merkle Root Update Reduces User Claim Window in Bidding Projects
  
  
## Summary

The failure to extend `claimDeadline` in `updateProjectAllocations` after updating merkle roots causes a reduced claim window for users. The delay between `finalizeBids` and `updateProjectAllocations` (for off-chain calculations) consumes part of the intended window, leaving less time than intended for users to claim refunds or NFTs.

## Root Cause

In [`AlignerzVesting.sol:813-830`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L812-L829), `updateProjectAllocations` updates merkle roots and the refund root but does not update `claimDeadline`. The deadline is set in `finalizeBids` at line 804 as `block.timestamp + claimWindow`, and remains unchanged when allocations are updated later. Since `claimRefund` (line 838) and `claimNFT` (line 866) check `biddingProject.claimDeadline > block.timestamp`, the effective window is reduced by the time between `finalizeBids` and `updateProjectAllocations`.

```js
    function updateProjectAllocations(...)
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


## Internal Pre-conditions

1. Owner calls `finalizeBids` with placeholder merkle roots to close the project and set `claimDeadline = block.timestamp + claimWindow`
2. Backend performs off-chain calculations (e.g., ~5 minutes)
3. Owner calls `updateProjectAllocations` to set the correct merkle roots
4. `claimDeadline` remains unchanged from step 1

## External Pre-conditions

None

## Attack Path

1. Owner calls `finalizeBids(projectId, refundRoot, merkleRoots, claimWindow)` at time `T`, setting `claimDeadline = T + claimWindow`
2. Backend calculates refund amounts and TVS allocations off-chain (e.g., ~5 minutes)
3. Owner calls `updateProjectAllocations(projectId, refundRoot, merkleRoots)` at time `T + delay` to set correct merkle roots
4. `updateProjectAllocations` does not extend `claimDeadline`
5. Users can only claim between `T + delay` and `T + claimWindow`, reducing the window by `delay`

## Impact

Users have less time to claim refunds or NFTs than intended. The window is reduced by the delay between `finalizeBids` and `updateProjectAllocations`. If the delay is significant relative to `claimWindow`, users may miss the deadline and lose access to refunds or NFT claims.

## Proof of Concept
N/A

## Mitigation

Extend `claimDeadline` in `updateProjectAllocations` to account for the delay.
  