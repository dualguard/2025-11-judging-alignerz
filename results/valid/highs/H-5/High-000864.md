# [000864] Cumulative Fee Applied Instead of Standard Fee, Reducing User Funds
  
  ### Summary

The `splitTVS()` and `mergeTVS()` functions internally rely on `calculateFeeAndNewAmountForOneTVS()` to apply fees to each element of the amounts array within an allocation. However, instead of applying the standard feeRate independently to each element, the function subtracts the cumulative fee from every array entry. As a result, users receive less than intended during split or merge operations, resulting in unintended loss of funds.

### Root Cause

In [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the logic incorrectly applies the accumulated fee total to each entry in the amounts array. Instead of deducting only the per-item fee, the function subtracts the cumulative fee amount, causing progressively larger and unintended reductions.

### Internal Pre-conditions

The allocation associated with the NFT contains multiple elements in the amounts array.

### External Pre-conditions

None

### Attack Path

None

### Impact

Users lose funds due to cumulative fee deduction rather than application of the standard feeRate.

### PoC

Add the following the to the `AlignerzVestingProtocolTest.t.sol` and run it via:
`forge test --mt testCummulativeFeeOnMergeTVS -vvvv`

```solidity
function testCummulativeFeeOnMergeTVS() external {
         vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

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
        // uint256[] memory newNfts;
        address bidder = bidders[0];
        uint256 poolId = bidderPoolIds[bidder];
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
        nftIds[0] = nftId;
        uint256[] memory percentages = new uint256[](5);
        for (uint8 i = 0; i < 5; i++)
        {
            percentages[i] = 2000;
        }
        vm.prank(bidder);
        (uint256 oldNft, uint256[] memory newNfts) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
        uint256[] memory firstRoundMerge = new uint256[](3);
        uint256[] memory firstRoundMergeProjects = new uint256[](3);
        for (uint8 i = 0; i < 3; i++)
        {
            firstRoundMerge[i] = newNfts[i];
            firstRoundMergeProjects[i] = PROJECT_ID;
        }
        uint256[] memory secondRoundMerge = new uint256[](1);
        uint256[] memory secondRoundMergeProjects = new uint256[](1);
        secondRoundMerge[0] = newNfts[3];
        secondRoundMergeProjects[0] = PROJECT_ID;

        vm.prank(projectCreator);
        vesting.setMergeFeeRate(200);
        vm.prank(bidder);
        nftId = vesting.mergeTVS(PROJECT_ID, nftId, firstRoundMergeProjects, firstRoundMerge);

        vm.prank(bidder);
        nftId = vesting.mergeTVS(PROJECT_ID, nftId, secondRoundMergeProjects, secondRoundMerge);
    }
```

```bash
...
emit TVSsMerged(projectId: 0, isBiddingProject: true, nftIds: 0x5ce5ef4a019c01725e091c2e114922171bc393129e8856935a83d761ed0f787e, mergedNftId: 1, amounts: [196000000000000000000 [1.96e20], 39200000000000000000 [3.92e19], 39200000000000000000 [3.92e19], 39200000000000000000 [3.92e19]], vestingPeriods: [7776000 [7.776e6], 7776000 [7.776e6], 7776000 [7.776e6], 7776000 [7.776e6]], vestingStartTimes: [1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0], claimedFlows: [false, false, false, false])
.....
emit TVSsMerged(projectId: 0, isBiddingProject: true, nftIds: 0x036b6384b5eca791c62761152d0c79bb0604c104a5fb6f4eb0703f3154bb3db0, mergedNftId: 1, amounts: [192080000000000000000 [1.92e20], 34496000000000000000 [3.449e19], 33712000000000000000 [3.371e19], 32928000000000000000 [3.292e19], 39200000000000000000 [3.92e19]], vestingPeriods: [7776000 [7.776e6], 7776000 [7.776e6], 7776000 [7.776e6], 7776000 [7.776e6], 7776000 [7.776e6]], vestingStartTimes: [1, 1, 1, 1, 1], claimedSeconds: [0, 0, 0, 0, 0], claimedFlows: [false, false, false, false, false])

Suite result: ok. 1 passed;
```

Note: Consider fixing the previous issues with `calculateFeeAndNewAmountForOneTVS()` function

### Mitigation

Modify `calculateFeeAndNewAmountForOneTVS()` so that only the single-element fee, not the cumulative fee, is applied to each element
```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+     uint256 fee;
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
+          fee = calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;
+          newAmounts[i] = amounts[i] - fee;
+          feeAmount += fee;
        }
    }
```
  