# [000191] Off-by-one error in `createPool` allows the creation of 11 pools which breaks protocol invariant.
  
  ### Summary

The `createPool` function in _AlignerzVesting_ contract allows the owner to create a new pool for a given `biddingProject`. One of the invariants of the protocol is that a maximum of 10 pools can be created per `biddingProject`.

The problem is that the current implementation of `createPool` allows the creation of up to 11 pools, breaking the protocol invariant.

### Root Cause

The root cause lies in the `createPool` function: 

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
        // @audit LOW  pool count can be 11
>     require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());

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

The line ` require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());` allows `poolCount` to be 10, and its value will be incremented to 11 at the end of the function.

### Attack Path

No specific attack path, this is just a logical off-by-one error allowing the owner to create 11 pools.

### Impact

The impact of this issue can be considered low as it results in a protocol invariant broken. The `createPool` function can only be called by the owner so the impact is limited.

### PoC

Please copy paste the following test in _AlignerzVestingProtocolTest_. test contract:

```solidity
    function test_CanCreate11Pools() public {
        vm.startPrank(projectCreator);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );

        // Create 11 pools (should only allow 10 according to invariant)
        for (uint256 i = 0; i < 11; i++) {
            vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, false);
            console.log("Created pool number", i + 1);
        }

        // Attempting to create a 12th pool should fail
        vm.expectRevert();
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, false);

        vm.stopPrank();
    }
```

The test highlights the creation of 11 pools for a `biddingProject`.



### Mitigation

Modify the `createPool` function to only allow the creation of 10 pools:

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
        // @audit LOW  pool count can be 11
>     require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());

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
  