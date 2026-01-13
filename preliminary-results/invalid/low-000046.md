# [000046] Pool Count Check Allows 11 Pools instead of 10
  
   
In `AlignerzVesting::createPool()`, the `require` statement intended to restrict an owner from creating more than **10 pools** is incorrectly implemented. The current check:

```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

still allows the creation of an **11th pool**. This happens because the condition uses `<=` instead of the correct `<`.
When `poolCount == 10`, the check passes, enabling one additional pool to be created.

**Note** : the pool count started from `zero` not from `1 `
```solidity
 biddingProject.poolCount = 0;
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

### Impact

* The owner can create **11 pools** instead of the intended maximum of 10.
* This means the owner must also supply **11 Merkle roots**, contradicting the system’s stated limit.
* The contract claims a maximum of 10 pools, but the logic silently violates that rule.

 
###  Recommendation 

Use a strict `< 10` check to ensure the pool count never exceeds the intended limit:

```diff
function createPool(
    uint256 projectId,
    uint256 totalAllocation,
    uint256 tokenPrice,
    bool hasExtraRefund
)
    external
    onlyOwner
{
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    require(totalAllocation > 0, Zero_Value());
    require(tokenPrice > 0, Zero_Value());

    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(!biddingProject.closed, Project_Already_Closed());
-   require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+   require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());

    biddingProject.token.safeTransferFrom(msg.sender, address(this), totalAllocation);

    uint256 poolId = biddingProject.poolCount;
    biddingProject.vestingPools[poolId] = VestingPool({
        merkleRoot: bytes32(0), // Initialize with empty Merkle root
        hasExtraRefund: hasExtraRefund
    });

    biddingProject.poolCount++;

    emit PoolCreated(projectId, poolId, totalAllocation, tokenPrice, hasExtraRefund);
}
```
 
  