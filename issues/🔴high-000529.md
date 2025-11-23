# [000529] Inconsistent State During Project Updates Allows Unfair Token Distribution
  
  ### Summary

The `updateProjectAllocations` function allows the project owner to modify merkle roots, but if some users have already claimed their NFTs, it creates inconsistent allocation states where users with identical bids receive different token amounts based on when they claimed.

### Root Cause


https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L812
The `AlignerzVesting::updateProjectAllocations` function overwrites merkle roots without migrating existing allocations, creating permanent state inconsistencies between users who claimed at different times.

### Internal Pre-conditions

1. A bidding project must be finalized with initial merkle roots
2. Some users must have already claimed NFTs using the initial merkle proofs
3. Project owner must call `updateProjectAllocations` with new merkle roots

### External Pre-conditions

No response

### Attack Path

No response

### Impact

- Unfair token distribution between users with identical bids
- Economic manipulation where owner can favor/disfavor specific users

### PoC


```javascript
    function test_InconsistentStateDuringProjectUpdates_Vulnerability() public {
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
    vesting.createPool(PROJECT_ID, 10_000_000 ether, 0.01 ether, true);
    vesting.addUsersToWhitelist(bidders, PROJECT_ID);
    vm.stopPrank();

    // 2. Place identical bids from multiple users
    for (uint256 i = 0; i < 5; i++) {
        vm.startPrank(bidders[i]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 180 days);
        vm.stopPrank();
    }

    // 3. Prepare initial bid allocations (fair distribution)
    BidInfo[] memory initialBids = new BidInfo[](5);
    for (uint256 i = 0; i < 5; i++) {
        initialBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD, // Everyone gets same allocation initially
            vestingPeriod: 180 days,
            poolId: 0,
            accepted: true
        });
    }

    // 4. Generate initial merkle roots (fair distribution)
    bytes32[] memory initialPoolRoots = new bytes32[](1);
    initialPoolRoots[0] = generateMerkleProofs(initialBids, 0);
    bytes32 initialRefundRoot = bytes32(0); // No refunds initially

    // 5. Finalize project with initial fair allocations
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, initialRefundRoot, initialPoolRoots, 60);

    // 6. First 3 users claim NFTs with initial fair allocations
    uint256[] memory initialNFTs = new uint256[](3);
    for (uint256 i = 0; i < 3; i++) {
        vm.prank(bidders[i]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[i]]);
        initialNFTs[i] = nftId;
        bidderNFTIds[bidders[i]] = nftId;
        
        // Verify NFT was minted and owned by bidder
        assertEq(nft.ownerOf(nftId), bidders[i], "NFT should be owned by bidder");
    }

 
    BidInfo[] memory updatedBids = new BidInfo[](5);
    for (uint256 i = 0; i < 5; i++) {
        uint256 favoredAmount = (i < 3) ? BIDDER_USD : BIDDER_USD * 2; // Double allocation
        updatedBids[i] = BidInfo({
            bidder: bidders[i],
            amount: favoredAmount,
            vestingPeriod: 180 days,
            poolId: 0,
            accepted: true
        });
    }

    // 8. Generate updated merkle roots (unfair distribution)
    bytes32[] memory updatedPoolRoots = new bytes32[](1);
    updatedPoolRoots[0] = generateMerkleProofs(updatedBids, 0);

    // VULNERABILITY TRIGGER: Update project allocations after some users already claimed
    vm.prank(projectCreator);
    vesting.updateProjectAllocations(PROJECT_ID, initialRefundRoot, updatedPoolRoots);

    // 9. Remaining 2 users claim NFTs with updated (favored) allocations
    for (uint256 i = 3; i < 5; i++) {
        vm.prank(bidders[i]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD * 2, bidderProofs[bidders[i]]);
        bidderNFTIds[bidders[i]] = nftId;
        
        // Verify NFT was minted and owned by bidder
        assertEq(nft.ownerOf(nftId), bidders[i], "NFT should be owned by bidder");
    }

    // 10. VERIFY VULNERABILITY: Check through actual token claims instead of allocationOf mapping
    vm.warp(block.timestamp + 90 days); // 50% vested

    // Early user (bidder0) claims tokens
    uint256 earlyUserBalanceBefore = token.balanceOf(bidders[0]);
    vm.prank(bidders[0]);
    vesting.claimTokens(PROJECT_ID, initialNFTs[0]);
    uint256 earlyUserBalanceAfter = token.balanceOf(bidders[0]);
    uint256 earlyUserClaimed = earlyUserBalanceAfter - earlyUserBalanceBefore;

    // Late user (bidder3) claims tokens  
    uint256 lateUserBalanceBefore = token.balanceOf(bidders[3]);
    vm.prank(bidders[3]);
    vesting.claimTokens(PROJECT_ID, bidderNFTIds[bidders[3]]);
    uint256 lateUserBalanceAfter = token.balanceOf(bidders[3]);
    uint256 lateUserClaimed = lateUserBalanceAfter - lateUserBalanceBefore;

    // PROOF: Late user received 2x more tokens than early user despite identical bids
    assertTrue(lateUserClaimed > earlyUserClaimed, "Vulnerability confirmed: inconsistent rewards");
    assertEq(lateUserClaimed, earlyUserClaimed * 2, "Late user should get exactly 2x more tokens");
}
```

### Mitigation

Remove the ability to update allocations after users have started claiming, or implement a migration mechanism that ensures consistency.
  