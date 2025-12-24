# [000608] Pools per project can exceed 10
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

Incorrect require max pool implementation when creating a pool will cause the pools to exceed 10

### Root Cause

In AlignerzVesting, 
Currently when creating pools the protocol requires that the number of pools per project should not exceed 10.

```javascript
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

        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++; //  @audit will allow creating next pool even 11

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }
```
Lets look at this require statement below which tries to prevent adding more than 10 pools per project.

```javascript
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project()); 
```
After a pool is added we increment the number of pools using the line below

```javascript
biddingProject.poolCount++;
```
The issue here is that the require statement uses `<= 10` which means that if `10 == 10` the condition still passes and later we increment it below with `biddingProject.poolCount++;` which means 11 pools might be created breaking the invariant.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Owner calls `createPool` function to create pool, at this time 10 pools are already added to the project, because of this `<= 10` . If we are already in the 10th pool 10 == 10 will pass making the owner to add the 11th pool

### Impact

A project could exceed 10 pools which is the max

### PoC

_No response_

### Mitigation

use strictly < to make sure the pools are always max 10 . so the line should be 

```javascript
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project()); 
```

  