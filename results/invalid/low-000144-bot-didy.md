# [000144] Owner can create an 11th pool despite 10-pool cap
  
  ### Summary

Off-by-one pool cap check (`<= 10` before increment) will allow a project to create 11 pools, violating the stated “max ten pools per project” invariant and potentially breaking off-chain assumptions.

### Root Cause

In AlignerzVesting.sol `createPool` function, the guard `require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project())` runs before increment; when `poolCount == 10`, the check passes, `poolId` is 10, and increment yields 11 total pools.

### Internal Pre-conditions

1. A bidding project exists and is open (`closed == false`).
2. Caller is owner and has approved enough tokens for pool allocations.
3. `poolCount` already 10 (pools 0–9 created).

### External Pre-conditions

none

### Attack Path

1. Owner calls `createPool` when `poolCount == 10`.
2. Guard `<= 10` passes; pool 10 is created; `poolCount` becomes 11.

### Impact

Project ends up with 11 pools (logic/invariant breach; low severity).

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test} from "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";

/// @dev PoC: Off-by-one in pool cap lets an 11th pool be created (poolCount <= 10 check).
contract PoolCapBugTest is Test {
    AlignerzVesting vesting;
    Aligners26 token;
    AlignerzNFT nft;
    MockUSD stable;

    function setUp() public {
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri/");
        token = new Aligners26("Aligners", "ALN"); // mints 26M to msg.sender
        stable = new MockUSD();

        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        vesting.setTreasury(address(1));
        nft.addMinter(proxy);

        // Launch a bidding project.
        vesting.launchBiddingProject(address(token), address(stable), block.timestamp, block.timestamp + 10 days, bytes32(0), false);

        // Approve enough tokens for 11 pools (11 * 1 ether).
        token.approve(address(vesting), 11 ether);
    }

    function test_OffByOneAllowsElevenPools() public {
        // Create 11 pools; the 11th should revert if cap were enforced, but passes due to <= check.
        for (uint256 i; i < 11; ++i) {
            vesting.createPool(0, 1 ether, 0.01 ether, false);
        }
        // Public getter for biddingProjects omits mappings; destructure to read poolCount (4th return value).
        (, , , uint256 poolCount, , , , , ,) = vesting.biddingProjects(0);
        assertEq(poolCount, 11, "poolCount should be 11 due to off-by-one cap");
    }
}
```

### Mitigation

Change guard to `poolCount < 10` (or pre-increment then compare) to enforce a hard cap of 10 pools.
  