# [000304] Using length array of allocation.amount to calculating fee on splitTVS() function, lead to splitTVS() function cannot be used
  
  ### Summary

In the `spliTVS()` function, split fee calculated below :

```solidity
   uint256[] memory amounts = allocation.amounts;
@> int256 nbOfFlows = allocation.amounts.length; //@audit-issue use length of array instead of array
   (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
```

The issue arises because `nbOfFlows` variable that using the length of `allocation.amounts` not the array, `length - 1`. This lead to, `calculateFeeAndNewAmountForOneTVS` always revert because try to access non-existing array.

Note

This occur on [mergeTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1011-L1013) function too.
Thus, the `mergeTVS()` function also cannot be used.

### Root Cause

In [AlignerzVesting.sol:1068-1069](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1068-L1069) use length of array to calculate split fee

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

Very straightforward, because of this issue, `splitTVS()` can’t be used from the start.

### Impact

`splitTVS()` function cannot be used

### PoC

Add test to `AlignerzVestingProtocolTest.t.sol` and run `forge test --match-test testSplitTVS`

```solidity
function testSplitTVS() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

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
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
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
        uint256[] memory nftIds = new uint256[](15);
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            vm.prank(bidder);
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 10. Some users try to claim refunds
        for (uint256 i = 15; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[bidder]);

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }

        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 60 days);

        // preparing to split
        address bidder = bidders[0];
        uint256[] memory percentage = new uint256[](2);
        percentage[0] = 5000;
        percentage[1] = 5000;
        vm.prank(bidder);
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentage, 1);

    }

```

Result :

```solidity
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] testSplitTVS() (gas: 19441685)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.55s (14.67ms CPU time)

Ran 1 test suite in 2.56s (2.55s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Change logic to :

```diff
   uint256[] memory amounts = allocation.amounts;
-- int256 nbOfFlows = allocation.amounts.length; 
++ int256 nbOfFlows = allocation.amounts.length - 1; 
   (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);

```
  