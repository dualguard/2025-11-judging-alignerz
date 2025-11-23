# [000208] Cumulative Fee Subtraction in calculateFeeAndNewAmountForOneTVS Causes User Overcharge and Token Loss
  
  ### Summary

In FeesManager.sol:169, the cumulative feeAmount is subtracted from each flow instead of the individual fee for that flow, which will cause permanent token loss and overcharging for TVS holders as any user performing split or merge operations will have tokens deducted beyond the intended fee rate, with the excess becoming stuck in the contract.

### Root Cause

In FeesManager.sol:169 the subtraction uses the cumulative feeAmount variable instead of the individual fee calculated for that specific flow:

``` solidity
  newAmounts[i] = amounts[i] - feeAmount; // feeAmount is cumulative, not per-flow
```

### Internal Pre-conditions

1. splitFeeRate or mergeFeeRate needs to be set to any value greater than 0
2. User's TVS allocation needs to have at least 2 flows in the amounts array

### External Pre-conditions

None required.


### Attack Path

1. User calls `splitTVS()` with a TVS that has multiple flows (e.g., 3 flows with amounts [1000e18, 1000e18, 1000e18])
2. calculateFeeAndNewAmountForOneTVS() is called with feeRate = 100 (1%)
3. Loop iteration 0: feeAmount = 10e18, newAmounts[0] = 990e18 (correct)
4. Loop iteration 1: feeAmount = 20e18, newAmounts[1] = 1000e18 - 20e18 = 980e18 (should be 990e18)
5. Loop iteration 2: feeAmount = 30e18, newAmounts[2] = 1000e18 - 30e18 = 970e18 (should be 990e18)
6. Treasury receives 30e18 tokens as fee
7. User loses 60e18 tokens total (10 + 20 + 30) but only 30e18 goes to treasury


### Impact

TVS holders suffer approximately 2x the intended fee loss when performing split/merge operations on multi-flow TVSs. For a 3-flow TVS with 1% fee rate, users lose 2% instead of 1%, with half becoming permanently stuck in the contract. The protocol also suffers from permanent accounting drift where tokens are locked without attribution to any user or treasury.

### PoC

```solidity
    function test_SplitFlow() public {
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
            true
        );

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
            usdt.approve(address(vesting), BIDDER_USD + 1_000 ether);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1)
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
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1)
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
        uint256[] memory percentages = new uint256[](4);
        percentages[0] = 2500;
        percentages[1] = 2500;
        percentages[2] = 2500;
        percentages[3] = 2500;

        //split
        vm.startPrank(bidders[0]);
        uint256 poolId = bidderPoolIds[bidders[0]];
        uint256 nftId = vesting.claimNFT(
            PROJECT_ID,
            poolId,
            BIDDER_USD,
            bidderProofs[bidders[0]]
        );
        // Expect Revert
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
```

### Mitigation

``` solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length); // Initialize array
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee; // Subtract individual fee, not cumulative
        unchecked { ++i; } // Add increment
    }
}
```
  