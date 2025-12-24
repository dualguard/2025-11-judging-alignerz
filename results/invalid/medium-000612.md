# [000612] Eleven pools can be created instead of the enforced 10
  
  ### Summary

The contract enforces that `poolCount` cannot exceed 10, but fails to implement it. 11 pools can be created.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689

### Details

In `createPool` function, we see that the protocol attempts to cap the amount of pools at 10: 

```jsx
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

However, if we look at how the poolCount is calculated here:

```jsx
     uint256 poolId = biddingProject.poolCount;
        biddingProject.vestingPools[poolId] = VestingPool({
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            hasExtraRefund: hasExtraRefund
        });

        biddingProject.poolCount++;
```

The first poolId will be 0, as array indexes start from 0. After getting 0 as the first pool, the poolCount will be incremented via `biddingProject.poolCount++;` . 

Which means, on the 2nd pool creation, poolCount will be 1, on the 3rd pool creation, poolCount will be 2 and so on and so on.

When poolCount reaches 10, there will be 11 pools have been created (since array indexes start from 0)

### Impact
High likelihood and low impact, therefore medium.

This issue goes against the intended behavior of the contract, hence I mark this issue as medium severity.

### POC

```jsx
    function test_create_11_pools() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );

        for (uint256 i = 0; i < 11; i++) {
            vesting.createPool(PROJECT_ID, 1 ether, 0.01 ether, true);
        }

        
        (,,, uint256 poolCount,,,,,,) = vesting.biddingProjects(PROJECT_ID);
        console.log("Pools created: ", poolCount);
    }
```
Terminal output:

```jsx
 [0] console::log("Pools created: ", 11) [staticcall]

```
  