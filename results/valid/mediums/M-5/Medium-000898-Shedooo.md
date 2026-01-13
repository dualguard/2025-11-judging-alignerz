# [000898] A holder of mixed-start TVS flows will have claims blocked by arithmetic panic
  
  ### Summary

The unchecked subtraction `block.timestamp - vestingStartTime` in `getClaimableAmountAndSeconds` will cause an arithmetic panic for NFT holders as a holder calls `claimTokens` while any flow starts in the future, blocking all claims (liveness failure) for any mixed-start TVS

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol` at `getClaimableAmountAndSeconds` the code subtracts `block.timestamp - vestingStartTime` without guarding future-start flows, causing an underflow panic before the intended `No_Claimable_Tokens` guard and halting `claimTokens` for any allocation containing a future-start tranche.

### Internal Pre-conditions

1. A reward project or merge/split operation creates an allocation with a `vestingStartTime` in the future relative to `block.timestamp`.
2. The holder (or any user) calls `claimTokens` on an NFT that has at least one such future-start flow.

### External Pre-conditions

_No response_

### Attack Path

      1. User holds an NFT allocation with multiple flows, at least one of which starts in the future (created either directly via reward project claim or by merging a future-start TVS into an active one).
      2. User calls `claimTokens(projectId, nftId)`.
      3. The per-flow loop reaches the future-start tranche; `block.timestamp - vestingStartTime` underflows in `getClaimableAmountAndSeconds`, causing a panic.
      4. The entire claim reverts, blocking withdrawal of already-accrued flows.

### Impact

NFT holders cannot withdraw already-vested tokens if any future-start flow exists in the same allocation (complete liveness failure for those positions). No funds are stolen, but withdrawals are blocked until the last-starting flow begins

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "forge-std/StdError.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

/// @notice Reproduces the future-start underflow that blocks partial claims
contract AlignerzVestingBugReproTest is Test {
    MockUSD internal tvs;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal user = address(0xBEEF);

    uint256 internal constant ONE_TOKEN = 1_000_000; // 1 unit with 6 decimals

    function setUp() public {
        tvs = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        nft.addMinter(address(vesting));

        // Fund the vesting contract for reward allocations
        tvs.approve(address(vesting), type(uint256).max);
        stable.approve(address(vesting), type(uint256).max);

        // Treasury can be any non-zero address
        vesting.setTreasury(address(1));
    }

    /// @dev Claiming before vestingStartTime causes arithmetic underflow instead of a graceful 0-claim path.
    function test_Revert_WhenRewardFlowStartsInFuture() public {
        uint256 start = block.timestamp + 1 days;
        vesting.launchRewardProject(address(tvs), address(stable), start, 30 days);

        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = ONE_TOKEN;

        vesting.setTVSAllocation(0, ONE_TOKEN, 30 days, kols, amounts);

        vm.prank(user);
        vesting.claimRewardTVS(0);

        uint256 nftId = 1; // first minted token id (ERC721A starts at 1)
        assertEq(nft.ownerOf(nftId), user);

        vm.prank(user);
        vm.expectRevert(stdError.arithmeticError);
        vesting.claimTokens(0, nftId);
    }

    /// @dev Merging a future-start flow with an active flow freezes all claims because the loop hits the underflowing tranche.
    function test_Revert_WhenMergedFutureFlowBlocksActiveFlow() public {
        // Flow A: already started 30 days ago
        uint256 startPast = block.timestamp - 30 days;
        vesting.launchRewardProject(address(tvs), address(stable), startPast, 120 days);
        address[] memory kolA = new address[](1);
        kolA[0] = user;
        uint256[] memory amtA = new uint256[](1);
        amtA[0] = ONE_TOKEN;
        vesting.setTVSAllocation(0, ONE_TOKEN, 60 days, kolA, amtA);

        // Flow B: starts in 30 days
        uint256 startFuture = block.timestamp + 30 days;
        vesting.launchRewardProject(address(tvs), address(stable), startFuture, 120 days);
        address[] memory kolB = new address[](1);
        kolB[0] = user;
        uint256[] memory amtB = new uint256[](1);
        amtB[0] = ONE_TOKEN;
        vesting.setTVSAllocation(1, ONE_TOKEN, 60 days, kolB, amtB);

        vm.prank(user);
        vesting.claimRewardTVS(0); // nftId 1
        vm.prank(user);
        vesting.claimRewardTVS(1); // nftId 2

        uint256 nftIdActive = 1;
        uint256 nftIdFuture = 2;

        // Merge future-start flow into the active NFT
        uint256[] memory mergeProjectIds = new uint256[](1);
        mergeProjectIds[0] = 1;
        uint256[] memory mergeNftIds = new uint256[](1);
        mergeNftIds[0] = nftIdFuture;

        vm.prank(user);
        vesting.mergeTVS(0, nftIdActive, mergeProjectIds, mergeNftIds);

        // Attempting to claim while the second flow hasn't started hits the underflow in getClaimableAmountAndSeconds
        vm.expectRevert();
        vm.prank(user);
        vesting.claimTokens(0, nftIdActive);
    }
}
```

### Mitigation

      - Clamp elapsed time: if `block.timestamp <= vestingStartTime`, return (0,0); if `block.timestamp >= vestingStartTime + vestingPeriod`, cap at `vestingPeriod`.
      - Use saturating subtraction: `claimableSeconds = secondsPassed > claimedSeconds ? secondsPassed - claimedSeconds : 0`.
      - Move the “nothing claimable” revert to an aggregate check after the flows loop so already-started flows can be claimed even when others aren’t yet claimable.
  