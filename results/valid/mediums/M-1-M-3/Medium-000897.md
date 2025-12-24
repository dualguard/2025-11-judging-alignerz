# [000897] Integrators will mis-account user vesting because the mirror stays stale
  
  ### Summary

The public mirror `allocationOf` never updates after claims or merges, causing stale vesting data for integrators and users as any consumer will read `allocationOf` and mis-account the vesting progress/amounts compared to the authoritative project storage.

### Root Cause

 In `protocol/src/contracts/vesting/AlignerzVesting.sol:941-1063` the public mirror `allocationOf[nftId]` is only written at NFT creation and split time, while mutating paths `claimTokens` and `mergeTVS` update only the authoritative project allocations (rewardProjects[..].allocations / biddingProjects[..].allocations) and never refresh `allocationOf[nftId]`, leaving the mirror stale.

### Internal Pre-conditions

1. A reward or bidding project exists and has minted vesting NFTs (TVS).
2. A holder performs a `claimTokens` or merges NFTs via `mergeTVS`, which mutates the authoritative allocation but not the mirror.

### External Pre-conditions

none.

### Attack Path

1. User (or any holder) calls `claimTokens` on their NFT; the contract updates claimedSeconds/claimedFlows/isClaimed only in project allocations.
2. Alternatively, user merges NFTs via `mergeTVS`, which fee-shaves and appends flows to the authoritative allocation.
3. The public mirror `allocationOf[nftId]` remains at its original snapshot, so any reader using it sees outdated amounts, flow geometry, and claimed progress.
4. Downstream consumers (frontends, analytics, on-chain helpers) that rely on `allocationOf` miscompute balances or payouts.

### Impact

 Integrators and users see wrong vesting progress/amounts and can mispay or be mispaid when relying on `allocationOf`. On-chain helpers like `A26ZDividendDistributor` that compute dividends from `allocationOf` will over- or under-distribute funds, creating real monetary loss or accounting drift.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

/// @dev Harness to surface the authoritative allocation stored inside project mappings.
contract AlignerzVestingHarness is AlignerzVesting {
    function exposedRewardAllocation(uint256 projectId, uint256 nftId) external view returns (Allocation memory) {
        return rewardProjects[projectId].allocations[nftId];
    }

    function exposedMirror(uint256 nftId) external view returns (Allocation memory) {
        return allocationOf[nftId];
    }
}

/// @notice Demonstrates that allocationOf mirror never updates after mutations (claim / merge),
///         leaving external readers with stale vesting progress.
contract AlignerzAllocationMirrorDesyncTest is Test {
    MockUSD internal tvs;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVestingHarness internal vesting;

    address internal user = address(0xBEEF);

    uint256 internal constant ONE_TOKEN = 1_000_000; // 1 unit with 6 decimals

    function setUp() public {
        tvs = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVestingHarness();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));

        // Fund vesting for allocations
        tvs.approve(address(vesting), type(uint256).max);
        stable.approve(address(vesting), type(uint256).max);

        // Treasury can be any non-zero address
        vesting.setTreasury(address(1));
    }

    function test_MirrorStale_AfterClaims() public {
        // Vesting starts now; 30-day linear vest of 1 token to user
        uint256 start = block.timestamp;
        vesting.launchRewardProject(address(tvs), address(stable), start, 60 days);

        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = ONE_TOKEN;
        vesting.setTVSAllocation(0, ONE_TOKEN, 30 days, kols, amounts);

        // User mints the vesting NFT
        vm.prank(user);
        vesting.claimRewardTVS(0);
        uint256 nftId = 1; // ERC721A starts at 1

        // Warp 15 days into vesting and make a partial claim
        vm.warp(start + 15 days);
        vm.prank(user);
        vesting.claimTokens(0, nftId);

        // Authoritative state advanced; mirror stayed at creation snapshot
        AlignerzVesting.Allocation memory realAlloc = vesting.exposedRewardAllocation(0, nftId);
        AlignerzVesting.Allocation memory mirrorAlloc = vesting.exposedMirror(nftId);

        assertGt(realAlloc.claimedSeconds[0], 0, "real claimedSeconds should advance");
        assertEq(mirrorAlloc.claimedSeconds[0], 0, "mirror remains at zero after claim");

        // Finish vesting to flip isClaimed in real storage
        vm.warp(start + 40 days);
        vm.prank(user);
        vesting.claimTokens(0, nftId);

        realAlloc = vesting.exposedRewardAllocation(0, nftId);
        mirrorAlloc = vesting.exposedMirror(nftId);

        assertTrue(realAlloc.isClaimed, "authoritative record says fully claimed");
        assertFalse(mirrorAlloc.isClaimed, "mirror never marked claimed");
    }
}
```

### Mitigation

Refresh `allocationOf[nftId]` after any mutation that changes the authoritative allocation: at least in `claimTokens` and after `mergeTVS` completes (and any other writer, e.g., future functions). Alternatively, remove the mirror and always read per-project allocations to avoid duplication and drift.
  