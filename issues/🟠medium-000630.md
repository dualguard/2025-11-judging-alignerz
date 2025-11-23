# [000630] Owner will exceed documented pool cap and create an unintended extra pool
  
  ### Summary

The incorrect boundary condition `require(biddingProject.poolCount <= 10)` in `createPool ` will cause a silent violation of the intended pool limit for the protocol as the owner will create an `11th pool (ID 10),` potentially breaking assumptions in off-chain allocation tooling.

### Root Cause

In `AlignerzVesting.sol:689 (approx; createPool)`, 

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

The validation uses` <= 10` instead of `< 10`. Because` pool IDs`` start at `0 `and increment post-use, `pool IDs 0–10 `inclusive are allowed `(11 pools total), conflicting with the semantic error reported `(Cannot_Exceed_Ten_Pools_Per_Project() implying max 10)`.

### Internal Pre-conditions

- Owner must have launched a bidding project `(projectId < biddingProjectCount)`.
- `biddingProject.closed == false`.
- `poolCount` has reached 10 (after creating pools 0→9).
- Owner has sufficient tokens approved for one more `totalAllocation.`

### External Pre-conditions

None

### Attack Path

- Owner calls` createPool(projectId, ...)` repeatedly until `poolCount == 10 (pools 0–9 created)`.
- Owner calls `createPool(projectId, ...) `again; check passes because `10 <= 10`.
- `Pool `with `ID 10` is created; `poolCount becomes 11.`

### Impact

- Off-chain scripts sized for 10 pools may truncate or misassign pool roots.
- Unexpected extra pool may dilute token distribution or break front-end assumptions.

### PoC

Add this  function to the `AlignerzVestingProtocolTest.t.sol`

```solidity
// ---
    // ---  Pool Count Off-by-One Error Allows 11 Pools  ---
    // ---
    // This test demonstrates that the pool count validation allows 11 pools (0-10) instead of 10,
    // violating the protocol's stated design and potentially causing array bound issues.
    function test_POC_PoolCount_OffByOne_Allows11Pools() public {
        vm.startPrank(projectCreator);
        
        // Launch a bidding project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1000,
            "0x0",
            false
        );

        console.log("=== TESTING POOL COUNT VALIDATION ===");
        
        // Create pools 0 through 9 (should succeed - 10 pools total)
        for (uint256 i = 0; i < 10; i++) {
            vesting.createPool(PROJECT_ID, 100_000 ether, 0.01 ether, false);
            console.log("Created pool ID:", i);
        }

        // CRITICAL: The 11th pool (poolId = 10) should be rejected but ISN'T
        // The require statement is: poolCount <= 10 (allows up to poolCount = 10)
        // This means we can create pool IDs 0,1,2,3,4,5,6,7,8,9,10 = 11 POOLS!
        
        console.log("Attempting to create 11th pool (poolId 10)...");
        
        // This should fail with "Cannot_Exceed_Ten_Pools_Per_Project" but it DOESN'T
        // because the check is "<=" instead of "<"
        vesting.createPool(PROJECT_ID, 100_000 ether, 0.01 ether, false);
        console.log("SUCCESS: Created 11th pool (poolId 10) - BUG CONFIRMED");

        // Verify we now have 11 pools
        (, , , uint256 poolCount_, , , , , , ) = vesting.biddingProjects(PROJECT_ID);
        assertEq(poolCount_, 11, "Pool count should be 11, not 10");
        
        console.log("Final pool count:", poolCount_);
        console.log("=== BUG CONFIRMED: 11 pools allowed instead of 10 ===");

        // NOW trying to create a 12th pool should fail
        vm.expectRevert(AlignerzVesting.Cannot_Exceed_Ten_Pools_Per_Project.selector);
        vesting.createPool(PROJECT_ID, 100_000 ether, 0.01 ether, false);
        
        vm.stopPrank();
    }
```

### Mitigation

- Change validation to `require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());`.
  