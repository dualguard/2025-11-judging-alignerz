# [000519] [L] Off by one error while creating pools in `createPool()` function
  
  ### Summary

In contract AlignerzVesting.sol the creator is allowed to create 10 pool but in createPool() function there is check which checks for number of pools like this: 
`require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());` 
Here because of the `<=` symbol the 10 is also considered while creating pool which results in creation of 11th pool.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-aga7hokakological/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L689


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- Bidding project is created 
- then creator can directly call createPool() 11 times

### Impact

- Off by one limit exceeds (mismatch between architecture and actual implementation)
- Allows creator to create one more pool 

### PoC

```solidity
function test_MoreThanTenPoolsCanBeCreated() external {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.03 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.03 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.03 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, false);
    }
```

### Mitigation

To resolve the issue change the line to: `require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());` 
  