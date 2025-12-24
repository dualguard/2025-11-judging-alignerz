# [000397] Off-By-One Error Allows Creation of 11 Pools Instead of Maximum 10
  
  ### Summary

The protocol intends to enforce a strict limit of maximum 10 pools per project.
However, due to an off-by-one condition:

```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

the contract allows creation of 11 pools (pool IDs 0 through 10), violating protocol design assumptions. Many functions also use this check. It can also cause problems.

```solidity
merkleRoots.length == biddingProject.poolCount
```

### Root Cause

The requirement uses:

```solidity
biddingProject.poolCount <= 10
```

This check passes for poolCount == 10, allowing the creation of a pool with ID 10 → the 11th pool.

The intended condition should be:

```solidity
biddingProject.poolCount < 10
```

Because pools are indexed from 0, the correct maximum valid index is 9.


### Impact

1. Loss of Funds: The tokens transferred into the 11th pool may become permanently unclaimable if no other function processes that pool ID.
2. Broken UI / Incorrect User Allocations: Off-chain components expecting a max 10 pools will ignore the extra one, causing:
 - Users missing refunds or TVS allocations
 - Incorrect total project allocation calculation
 - Unexpected or inconsistent claim status
3. Protocol Invariant Violation: The protocol clearly states maximum of 10 pools.
Once the invariant is violated. All assumptions around indexing break. All on-chain/off-chain code relying on pool count becomes unreliable

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {AlignerzVestingProtocolTest} from "test/AlignerzVestingProtocolTest.t.sol";

contract buggy is AlignerzVestingProtocolTest {
    AlignerzVestingProtocolTest test;
    address user = makeAddr("user");
    
    function test_claimTVS() public {
        vm.deal(user, 50 ether);
        usdt.mint(user, BIDDER_USD);

        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        // Similar to the complete flow but with multiple projects
        // This demonstrates the contract's ability to handle multiple projects simultaneously

        // Setup first project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );

        // NOTE- project owner can create more than 10, that's what function wanted to prevent
        vesting.createPool(0, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.createPool(0, 3_000 ether, 0.01 ether, true);
        vesting.addUserToWhitelist(user, 0);
        vm.stopPrank();

        // Setup second project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp + 100, block.timestamp + 1_100_000, "0x0", true
        );
        vesting.createPool(1, 3_000_000 ether, 0.015 ether, false);
        vesting.addUserToWhitelist(user, 1);
        vm.stopPrank();

        vm.warp(block.timestamp + 100);

        // user will place his bid in first pool
        vm.startPrank(user);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(0, BIDDER_USD / 2, 90 days);
        vesting.updateBid(0, BIDDER_USD / 2, 90 days);
        vm.stopPrank();
    }
}
```
### Mitigation

Update the pool count check to enforce strict upper bound:

```diff
- require(biddingProject.poolCount <= 10, ...);
+ require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
  