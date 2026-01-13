# [000890] 10-pool cap bypass
  
  ### Summary

The off-by-one cap check in `createPool` will cause an extra (11th) pool to be creatable for bidding projects as the owner will call `createPool` when `poolCount` is 10 and the `<= 10` guard still permits the call.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:686-696` the guard uses `require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());`, so when `poolCount` is 10 the check still passes, allowing creation of a pool with id 10 (the 11th pool overall).

### Internal Pre-conditions

1. Owner (onlyOwner) needs to have already created 10 pools so that `poolCount` equals 10.
2. Owner calls `createPool` again on the same project while it is still open.

### External Pre-conditions

none

### Attack Path

1. Owner creates 10 pools for a bidding project, so `poolCount` becomes 10.
2. Owner calls `createPool` again; the `<= 10` guard passes with `poolCount == 10`.
3. A new pool with id 10 is created and `poolCount` increments to 11, exceeding the stated 10-pool cap.

### Impact

The protocol and integrators face inconsistent behavior with the advertised 10-pool limit; an 11th pool can be created, potentially breaking off-chain assumptions (e.g., Merkle root array sizing, UI limits) and leading to misconfiguration or operational errors. 

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";

/// @notice Reproduces the pool cap off-by-one: the 11th pool creation succeeds.
contract AlignerzPoolCapOffByOneTest is Test {
    AlignerzVesting internal vesting;
    AlignerzNFT internal nft;
    MockUSD internal token;
    MockUSD internal stable;

    uint256 internal constant ONE_TOKEN = 1_000_000; // 1 unit with 6 decimals

    function setUp() public {
        token = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));

        // Allow vesting to pull enough tokens for pool creation
        token.approve(address(vesting), type(uint256).max);
        stable.approve(address(vesting), type(uint256).max);

        vesting.setTreasury(address(1));

        uint256 start = block.timestamp;
        uint256 end = start + 1 days;
        vesting.launchBiddingProject(address(token), address(stable), start, end, keccak256("end"), false);
    }

    /// @dev With poolCount starting at 0 and the guard using <= 10, an 11th pool is allowed.
    function test_Revert_WhenCreatingEleventhPool() public {
        for (uint256 i; i < 10; i++) {
            vesting.createPool(0, ONE_TOKEN, 1, false);
        }

        // Current guard uses <= 10, so poolCount == 10 still passes and this call succeeds (off-by-one).
        vm.expectRevert(AlignerzVesting.Cannot_Exceed_Ten_Pools_Per_Project.selector);
        vesting.createPool(0, ONE_TOKEN, 1, false);
    }
}
```

### Mitigation

Change the guard in `createPool` to `require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());` so only pools 0–9 are allowed
  