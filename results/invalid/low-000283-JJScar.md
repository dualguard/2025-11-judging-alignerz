# [000283] `AlignerzVesting::createPool` Allows Creating of 11 Pools
  
  ### Summary

Each bidding project is allowed to have up to 10 pools. However, the check in the `createPool` function allows the creation of 11 pools. 

### Root Cause

Here is the `createPool` function snipped down to the relevant code:
```solidity
function createPool(
    uint256 projectId,
    uint256 totalAllocation,
    uint256 tokenPrice,
    bool hasExtraRefund
) external onlyOwner {
    // ... //
    require(
        biddingProject.poolCount <= 10, 
        Cannot_Exceed_Ten_Pools_Per_Project()
    );
    // ... //
}
```

The snippet shows the check allows the owner to create pools as long as the counter is below or equal to 10. This essentially means that if there are 10 pools or less, the function can continue. 

However, this is incorrect. The assumption is that any bidding project can have up to and including 10. 

If the 10th pool has been created, the counter will show 10. If the owner will call `createPool` again, the check will see that the counter is at 10, and will allow the creation of another pool. Making 11 pools possible. 

### Internal Pre-conditions

1. For this issue to happen 10 pools need to already exist for a single Bidding Project
2. Owner will have to call the `createPool`

### External Pre-conditions

_No response_

### Attack Path

This issue is not an attack per say. Which means that this will be an oversight that the contracts checks will allows.

### Impact

**Impact:** Medium <br>
The checks only allow one extra pool that deviates than the cap. This is no that harmful to either the protocol and the bidding project. But it still exceeds the max allowed.

**Likelihood:** Low <br>
This is owner controlled and will only happen for the 11th time. Not all projects will try and open many pools. 

### PoC

Paste the following in the `AlignerzVestingProtocolTest.t.sol` test suite for the runnable PoC:
```solidity
function test_11PoolsArePossible() public {
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vm.startPrank(projectCreator);
    vesting.launchBiddingProject(
        address(token),
        address(usdt),
        block.timestamp,
        block.timestamp + 1_000_000,
        "0x0",
        true
    );

    // 2. Create multiple pools with different prices
    uint256 count = 0; 
    token.approve(address(vesting), UINT256_MAX);
    for (uint256 i = 0; i <= 10; i++){ 
        vesting.createPool(PROJECT_ID, 10 ether, 0.01 ether, false);
        count++; // counting pools
    }
    assertEq(count, 11);
}
```

### Mitigation

Change the check to the following:
```diff
function createPool(
    uint256 projectId,
    uint256 totalAllocation,
    uint256 tokenPrice,
    bool hasExtraRefund
) external onlyOwner {
    // ... //
    require(
-       biddingProject.poolCount <= 10, 
+       biddingProject.poolCount < 10,
        Cannot_Exceed_Ten_Pools_Per_Project()
    );
    // ... //
}
```
  