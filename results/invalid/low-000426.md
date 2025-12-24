# [000426] ``Off by one`` error allows 11 pools creation instead of 10.
  
  ### Summary

``Off by one`` error allows 11 pools creation instead of 10.

### Root Cause

```solidity
    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(totalAllocation > 0, Zero_Value());
        require(tokenPrice > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  

        biddingProject.token.saf    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(totalAllocation > 0, Zero_Value());
        require(tokenPrice > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  

        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }eTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }
```
Here, 
```solidity
        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());   // <= instead of <

...
         uint256 poolId = biddingProject.poolCount; // starts from 0
...
        biddingProject.poolCount++;
...
```
It uses ``<=`` instead of ``<`` as ``poolCount`` starts from ``0``.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner sets the 11th pool.

### Impact

It is a owner protected function but it doesn't adhere to the ``invariant`` of setting only 10 pools. 

### PoC

_No response_

### Mitigation

Use ``<`` instead of ``<=``.
  