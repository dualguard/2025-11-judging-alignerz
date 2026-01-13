# [000803] State Mishandling in `splitTVS` Causes Permanent Loss of User Vested Tokens Through Storage Overwriting
  
  ### Summary

A state mishandling vulnerability in the `splitTVS` function will cause a permanent loss of vested tokens for NFT holders as the function incorrectly mutates the storage reference during iteration, leading to the original allocation being overwritten with fractional values and destroying up to 99%+ of the user's total vested amount.


### Root Cause

In `AlignerzVesting.sol:splitTVS` and `_computeSplitArrays`, the storage reference `allocation` is passed to `_computeSplitArrays` which reads from it during each iteration. However, this same storage reference is being mutated via `allocationOf[nftId] = newAlloc` where `newAlloc` points to the same storage location as `allocation`. This creates a feedback loop where:

1. The first iteration correctly calculates percentages based on the original amounts
2. Subsequent iterations read from the already-modified storage (now containing fractional values from the previous iteration)
3. Each iteration further reduces the amounts, compounding the loss
4. The final iteration reads from nearly depleted storage, especially when the highest percentage is last

The issue is exacerbated in `_assignAllocation` which updates `biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]`, and then this mutated storage is immediately assigned to `allocationOf[nftId]`, which aliases the same storage location being read from in the loop.

[Storing allocation](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1063-L1065)

[Passing allocation to _computeSplitArrays and overwritting](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088-L1092)

[_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

This is a vulnerability path (not requiring malicious intent):

1. User owns an NFT with vested token allocations worth 10,000 tokens
2. User calls `splitTVS` to split their NFT into 2 parts with percentages `[1, 9999]` (0.01% and 99.99%)
3. First iteration (i=0): Calculates 0.01% of 10,000 = 1 token, assigns to original NFT, and **mutates the storage allocation** to now contain this fractional amount
4. Second iteration (i=1): Reads from the **already mutated storage** which now shows ~1 token instead of 10,000, calculates 99.99% of ~1 = ~1 token
5. Both NFTs now hold only ~2 tokens total instead of 10,000 tokens
6. The remaining ~9,998 tokens are permanently lost and unclaimable by anyone
7. When user attempts to claim after vesting period, they receive less than 1% of their expected allocation


### Impact

The NFT holder suffers an approximate loss of 99%+ of their vested token allocation, with the exact percentage lost depending on the order of percentages in the split. The tokens become permanently locked in the contract as they are not assigned to any NFT. In the provided PoC with a 10,000 USDT allocation, the user loses over 9,900 USDT worth of vested tokens.

### PoC

```solidity
function test_Overwriting_State_User_Loses_POC() public {
    vm.startPrank(projectCreator);

    vesting.setVestingPeriodDivisor(1);

    // 1. Launch project
    vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

    // 2. Create multiple pools with different prices
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
    vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

    // 3. Place bids from different whitelisted users
    vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
    vm.stopPrank();

    vm.startPrank(bidders[0]);
    usdt.approve(address(vesting), BIDDER_USD);
    vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
    vm.stopPrank();

    // 4. Prepare bid allocations
    BidInfo[] memory allBids = new BidInfo[](2);

    for (uint256 i = 0; i < 2; i++) {
        allBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD,
            vestingPeriod: 60 days,
            poolId: 0,
            accepted: true
        });
    }

    // 6. Generate merkle roots for each pool
    bytes32[] memory poolRoots = new bytes32[](3);
    for (uint256 poolId = 0; poolId < 3; poolId++) {
        poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
    }

    // 7. Finalize project with merkle roots
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

    // 8. Users claim NFTs with proofs
    vm.startPrank(bidders[0]);
    uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);
    assertEq(nft.ownerOf(nftId), bidders[0]);
    vm.stopPrank();

    // 9. Splitting TVS to multiple NFTs
    vm.startPrank(bidders[0]);
    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 1;
    percentages[1] = 9_999;
    (uint256 oldNft, uint256[] memory newNft) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
    vm.stopPrank();

    // 10. Fast forward time to simulate vesting period
    vm.warp(block.timestamp + 90 days);

    // 11. Users claim tokens after vesting period
    uint256 tokenBalanceBefore = token.balanceOf(bidders[0]);
    vm.startPrank(bidders[0]);
    vesting.claimTokens(PROJECT_ID, oldNft);
    vesting.claimTokens(PROJECT_ID, newNft[0]);
    uint256 tokenBalanceAfter = token.balanceOf(bidders[0]);
    assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
    vm.stopPrank();
    uint256 bidderTokensClaimed = tokenBalanceAfter - tokenBalanceBefore;

    // The bidders token claims are far less than the bidder's total allocation (less than 1%)
    assertLt(bidderTokensClaimed, BIDDER_USD / 100);
}
```

### Mitigation

The root cause is that `_computeSplitArrays` reads from a storage reference that is being mutated during the loop. To fix this:

1. Cache the original allocation values in memory before the loop
2. Update `_computeSplitArrays` signature to accept memory instead of storage

This ensures all split calculations are based on the original allocation values, preventing the destructive feedback loop.
  