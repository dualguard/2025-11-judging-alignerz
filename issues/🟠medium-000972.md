# [000972] Users who have merged their TVS will not receive dividends.
  
  ### Summary

When the `mergeTVS` function is called, several TVSs are merged into one. Due to the lack of updates, the status in the `allocationOf` mapping, which updates unclaimed tokens in the Dividends contract, is not updated.

### Root Cause

When the [mergeTVS](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026) function is called, several TVSs are merged into one. In this function, the array of the main TVS token is increased. The remaining tokens are burned. The `mergeTVS` function updates the status of the main token in the `biddingProject` or `rewardProject` mapping. However, the [getTotalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) and [getUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) functions rely on the `allocationOf` mapping.

Due to the lack of an update to the status in the `allocationOf` mapping, which updates unclaimed tokens in the Dividends contract, this leads to a DoS, and the system will not calculate the number of unclaimed tokens at TVS and will not be able to accrue dividends. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user must merge several tokens into one.
2. Call the `getUnclaimedAmounts` function to update the `unclaimedAmountsIn` mapping so that dividends are credited to them based on this data. 

### Impact

This will result in a DoS for this token, meaning that the user will not receive dividends and will lose profits. It will also lead to a general DoS of the `getTotalUnclaimedAmounts` function.

### PoC

```solidity    
 function test_CompleteVestingFlow() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false
        );

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        // vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0)
                ? 90 days
                : (i % 3 == 1)
                    ? 180 days
                    : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

        // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0)
                    ? 90 days
                    : (i % 3 == 1)
                        ? 180 days
                        : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        // Claim NFTs for two different bidders from different pools
        address bidder0 = bidders[0]; // Will own both NFTs
        address bidder1 = bidders[1]; // Will transfer NFT to bidder0

        vm.prank(bidder0);
        uint256 nftId0 = vesting.claimNFT(
            PROJECT_ID,
            0, // poolId 0
            BIDDER_USD,
            bidderProofs[bidder0]
        );

        // Bidder 1 claims NFT from pool 1
        vm.prank(bidder1);
        uint256 nftId1 = vesting.claimNFT(
            PROJECT_ID,
            1, // poolId 1
            BIDDER_USD,
            bidderProofs[bidder1]
        );

        // Transfer NFT from bidder1 to bidder0
        vm.prank(bidder1);
        nft.transferFrom(bidder1, bidder0, nftId1);

        vm.startPrank(bidder0);
        uint256[] memory projectIds = new uint256[](1);
        uint256[] memory nftIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        nftIds[0] = nftId1;

        uint256 mergedNftId = vesting.mergeTVS(
            PROJECT_ID,
            nftId0,
            projectIds,
            nftIds
        );
        vm.expectRevert();
        dividendDistributor.getUnclaimedAmounts(mergedNftId);
    }
```

### Mitigation

Update the status in the `allocationOf` mapping in the `mergeTVS` function.
  