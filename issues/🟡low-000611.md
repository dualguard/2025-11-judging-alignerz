# [000611] finalizeBids will revert if any of the pools has merkleRoot set already
  
  ### Summary

`finalizeBids` function will revert if any of the pools has their merkleRoot set already, instead of skipping the already set pool.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L796

### Details

In `finalizeBids` there’s the following block:

```jsx
      for (uint256 poolId = 0; poolId < nbOfPools; poolId++) {
         =>   require(biddingProject.vestingPools[poolId].merkleRoot == bytes32(0), Merkle_Root_Already_Set());
            biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
            emit PoolAllocationSet(projectId, poolId, merkleRoots[poolId]);
        }
```

If one or more pools have their merkleRoot already set, for instance via `updateProjectAllocations` function, this call will revert instead of skipping the already set pool, because require statement reverts, doesn’t skip.

These are onlyOwner functions, and the impact is just revert and confusion. A better approach would be to skip the set pools, not revert the whole function. Hence, I mark this severity as low.
  