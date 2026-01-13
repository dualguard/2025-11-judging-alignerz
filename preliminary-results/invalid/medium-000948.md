# [000948] When AlignerzNFT.sol Pause functions is called it Blocks Vesting Claims While Tokens Continue Accruing
  
  ### Summary

Pausing AlignerzNFT.sol burn function to be specific causes fully vested allocations to become unclaimable, creating a liquidity bomb when unpaused.
When AlignerzNFT.sol burn function is paused, users with fully vested allocations cannot claim their tokens because the claimTokens(...) function attempts to burn the NFT, which reverts due to the pause this might seem as an intent of course for the pause. Meanwhile, vesting continues to accrue based on block.timestamp. When the contract is unpaused, all accumulated tokens release simultaneously in a concentrated liquidity event. Simply to say while the AlignerzNFT.sol burn function is paused the protocol shouldnt accrue 


### Root Cause

The claimTokens(...) function in AlignerzVesting.sol calls nftContract.burn(nftId) when an allocation is fully claimed. This burn function has the whenNotPaused modifier in AlignerzNFT.sol, meaning when the NFT contract is paused, all claims that would trigger a burn will revert.
Importantly vesting calculations in getClaimableAmountAndSeconds use block.timestamp independently of the pause state. This means vesting continues to accrue during the pause period, but users cannot claim once their vesting completes because the required burn operation is blocked.

> [vesting.](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941)

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L121](url)

```solidity
File: AlignerzVesting.sol
function claimTokens(uint256 projectId, uint256 nftId) external {
    // ... claim logic ...
    
    // Check if all flows are claimed
    if (allocation.claimedFlows == allocation.totalFlows) {
        nftContract.burn(nftId); //@audit reverts when NFT contract is paused
        allocation.isClaimed = true;
    }
}
```
```solidity
File: AlignerzVesting.sol
function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) 
    public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
    // ...
    if (block.timestamp > vestingPeriod + vestingStartTime) {
        secondsPassed = vestingPeriod; //@audit vesting continues regardless of pause
    } else {
        secondsPassed = block.timestamp - vestingStartTime;
    }
    // ...
}
```
```solidity
File: AlignerzNFT.sol
function burn(uint256 tokenId) public whenNotPaused {
    //@audit pausing blocks all claims of fully vested allocations
    _burn(tokenId, true);
}
```

### Internal Pre-conditions

1. Admin needs to call pause nft burn function in AlignerzNFT.sol
2. vesting still accrues while pause nft function but logically while paused since users are bricked to full claims the pause should also account that vesting uses block.timestamp meaning The vesting contract (AlignerzVesting.sol) has no awareness of the NFT contract's pause state. It continues calculating claimable amounts based on elapsed time while the NFT contract blocks the required burn operation for fully vested claims. This creates a synchronization failure between vesting accrual and claim finalization.

### External Pre-conditions

NONE

### Attack Path

none

### Impact

1.Users with fully vested allocations cannot claim tokens during pause period(we can consider this as the purpose of the pause function but its a little deny of service as we depend on the burn to fully claim as users)
2.Vesting calculations continue accruing based on block.timestamp regardless of pause state
3.Upon unpause, all users with completed vesting can claim simultaneously, creating synchronized selling pressure
4.The longer the pause, the more severe the liquidity bomb (days of accumulated vesting released at once)

### PoC

```solidity
function test_PauseLiquidityBomb() public {
        // Setup: Create a bidding project with standard configuration
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        
        vesting.launchBiddingProject(
            address(token), 
            address(usdt), 
            block.timestamp, 
            block.timestamp + 1_000_000, 
            "0x0", 
            true
        );
        
        vesting.createPool(PROJECT_ID, 10_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // Phase 1: Five large holders place significant bids
        address[] memory largeHolders = new address[](5);
        uint256 largeBidAmount = 10_000 ether;
        
        for (uint256 i = 0; i < 5; i++) {
            largeHolders[i] = bidders[i];
            usdt.mint(largeHolders[i], largeBidAmount);
            
            vm.startPrank(largeHolders[i]);
            usdt.approve(address(vesting), largeBidAmount);
            vesting.placeBid(PROJECT_ID, largeBidAmount, 180 days);
            vm.stopPrank();
        }

        // Phase 2: Prepare allocations and finalize
        BidInfo[] memory bids = new BidInfo[](5);
        for (uint256 i = 0; i < 5; i++) {
            bids[i] = BidInfo({
                bidder: largeHolders[i],
                amount: largeBidAmount,
                vestingPeriod: 180 days,
                poolId: 0,
                accepted: true
            });
        }
        
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(bids, 0);
        
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // Phase 3: Users claim their NFTs
        uint256[] memory nftIds = new uint256[](5);
        for (uint256 i = 0; i < 5; i++) {
            vm.prank(largeHolders[i]);
            nftIds[i] = vesting.claimNFT(
                PROJECT_ID, 
                0, 
                largeBidAmount, 
                bidderProofs[largeHolders[i]]
            );
        }

        // Phase 4: Time passes - 90 days into 180-day vesting (50% vested)
        vm.warp(block.timestamp + 90 days);

        // Phase 5: Users claim partial tokens (normal operation)
        uint256[] memory partialClaims = new uint256[](5);
        for (uint256 i = 0; i < 5; i++) {
            uint256 balanceBefore = token.balanceOf(largeHolders[i]);
            
            vm.prank(largeHolders[i]);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);
            
            uint256 balanceAfter = token.balanceOf(largeHolders[i]);
            partialClaims[i] = balanceAfter - balanceBefore;
        }

        // Phase 6: CRITICAL - Admin pauses NFT contract at day 90
        vm.prank(owner);
        nft.pause();

        // Phase 7: Time continues - vesting completes during pause (day 180 reached)
        vm.warp(block.timestamp + 90 days);

        // Phase 8: Users with fully vested allocations CANNOT claim during pause
        for (uint256 i = 0; i < 5; i++) {
            vm.prank(largeHolders[i]);
            vm.expectRevert(bytes4(keccak256("EnforcedPause()")));
            vesting.claimTokens(PROJECT_ID, nftIds[i]);
        }

        // Phase 9: Calculate expected totals
        // Each user already claimed partialClaims[i] at day 90 (50% of their allocation)
        // The remaining 50% should be claimable after vesting completes
        uint256 expectedRemainingPerUser = partialClaims[0]; // Remaining is equal to what was already claimed (50% left)
        uint256 totalExpectedBomb = expectedRemainingPerUser * 5;

        // Phase 10: UNPAUSE - The liquidity bomb trigger
        vm.prank(owner);
        nft.unpause();

        // Phase 11: Record state before synchronized mass claims
        uint256[] memory balancesBeforeBomb = new uint256[](5);
        for (uint256 i = 0; i < 5; i++) {
            balancesBeforeBomb[i] = token.balanceOf(largeHolders[i]);
        }

        // Phase 12: THE LIQUIDITY BOMB - All users claim simultaneously in one block
        uint256 totalBombRelease = 0;
        for (uint256 i = 0; i < 5; i++) {
            vm.prank(largeHolders[i]);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);
            
            uint256 balanceAfterBomb = token.balanceOf(largeHolders[i]);
            uint256 claimedInBomb = balanceAfterBomb - balancesBeforeBomb[i];
            totalBombRelease += claimedInBomb;
        }

        // Phase 13: Verify the liquidity bomb impact
        assertApproxEqRel(
            totalBombRelease,
            totalExpectedBomb,
            0.02e18,
            "Bomb did not release expected remaining tokens"
        );

        // The critical metric: tokens that should distribute over 90 days
        // instead released in ONE block
        uint256 normalDailyRelease = totalExpectedBomb / 90;
        uint256 daysWorthInOneBlock = totalBombRelease / normalDailyRelease;
        
        assertTrue(
            daysWorthInOneBlock >= 85,
            "Liquidity bomb should equal 85+ days of normal distribution"
        );
        
        // Verify NFTs were burned (allocation fully claimed)
        for (uint256 i = 0; i < 5; i++) {
            vm.expectRevert();
            nft.ownerOf(nftIds[i]);
        }

        emit log_string("\n=== LIQUIDITY BOMB IMPACT ===");
        emit log_named_uint("Total tokens released in single block", totalBombRelease / 1e18);
        emit log_named_uint("Days of normal vesting compressed", daysWorthInOneBlock);
        emit log_named_uint("Simultaneous claimers", 5);
        emit log_string("Result: 90 days of gradual selling pressure concentrated into 1 transaction");
    }
```

### Mitigation

Pause aware vesting calculation:
Track total pause duration and extend vesting periods accordingly to prevent accumulation during pause.
  