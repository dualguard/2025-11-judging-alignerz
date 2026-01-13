# [000180] `assignedPoolId` member of a user allocation will not reflect the real pool id for merged TVS.
  
  ### Summary

`assignedPoolId` is a field of the `Allocation` struct and is set in `claimNFT` function:

```solidity
    function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
        external
        returns (uint256)
    {
       ...

        biddingProject.allocations[nftId].assignedPoolId = poolId;   // @audit this allocation comes from poolId
        
       ....
    }
```

Because `assignedPoolId` is a field in `Allocation` struct, it is common to all flows contained in the struct. Hence, when a merge happens, all merged flows will have the same `assignedPoolId` as the NFT that will result from the merge (the initial one that is not burnt).

This means `assignedPoolId` doesn't reflect origin of flows but ideally, it should.

### Root Cause

`assignedPoolId`  should be an array of `uint256` , because it is specific to a given flow.

### Impact

The impact of this issue is low as it results in an incorrect internal state. An `Allocation` might look like the flows come from a given pool while most of them actually come from another one.

### Mitigation

Make `assignedPoolId`  an array of `uint256` and track it with each flow.
  