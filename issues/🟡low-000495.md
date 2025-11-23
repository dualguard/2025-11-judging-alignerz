# [000495] Method AlignerzVesting::createPool() allows 11 pools instead of 10 per projectId
  
  ### Summary

Method `AlignerzVesting::createPool()` allows 11 pools to be inserted by logic, but the error suggests `<= 10` should be allowed.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

### Root Cause

The poolCount starts with 0. If we include element id 10, then a total of 11 pools can be added.

```solidity
    /// @notice Creates a new vesting pool in a biddingProject
    /// @param projectId ID of the biddingProject
    /// @param totalAllocation Total tokens allocated to this pool
    /// @param tokenPrice token price set for this pool
    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(totalAllocation > 0, Zero_Value());
        require(tokenPrice > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());
@>        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  

        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }
```

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Protocol allows inclusion of 11 pools, While only 10 should be allowed.

### Impact

The implementation differ than the intended behaviour. 

### PoC

_No response_

### Mitigation

```diff
    /// @notice Creates a new vesting pool in a biddingProject
    /// @param projectId ID of the biddingProject
    /// @param totalAllocation Total tokens allocated to this pool
    /// @param tokenPrice token price set for this pool
    function createPool(uint256 projectId, uint256 totalAllocation, uint256 tokenPrice, bool hasExtraRefund)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(totalAllocation > 0, Zero_Value());
        require(tokenPrice > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());
++       require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());  
--        require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());  

        biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

        uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;

        emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
    }
```
  