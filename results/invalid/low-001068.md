# [001068] Off-by-One Error Allows Creation of 11 Pools Instead of the Intended Maximum of 10
  
  ### Summary

In `AlignerzVesting::createPool()`, the protocol intends to restrict each bidding project to a maximum of 10 pools.
However, the enforcement check uses:
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
This condition allows pool creation when `poolCount == 10`, meaning the contract will end up supporting 11 pools (0 through 10).

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

In the above line pool count check is done incorrectly 
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
This allows poolCount to reach 10 before blocking, enabling creation of the 11th pool.
The correct logic should be:
```solidity
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```


### Internal Pre-conditions

1. Owner must call `createPool()` repeatedly for the same projectId.
2. `biddingProject.poolCount` must reach 10.

### External Pre-conditions

_No response_

### Attack Path

1. Owner creates pools 0 through 9 -> poolCount = 10 (expected limit).
2. Owner calls `createPool()` again.
3. The check `poolCount <= 10` passes, allowing creation of pool 10 (the 11th pool).

### Impact

This bug allows the protocol to accidentally create 11 pools instead of the intended maximum of 10, breaking an important configuration rule of the bidding system.

### PoC

_No response_

### Mitigation

```diff
contract AlignerzVesting{
    .
    .
    .
    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(totalAllocation > 0, Zero_Value());
        require(tokenPrice > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());
-       require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  
+       require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());  
        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }
    .
    .
    .
}
```
  