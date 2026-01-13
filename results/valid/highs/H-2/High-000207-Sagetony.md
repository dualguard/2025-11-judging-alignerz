# [000207] Uninitialized Return Array in `calculateFeeAndNewAmountForOneTVS` Causes Complete DoS of Split and Merge Functions
  
  ### Summary

In `FeesManager.sol:169`, the newAmounts return array is never initialized with a length, which will cause a complete denial of service for all TVS holders as any user calling `splitTVS()` or` mergeTVS()` will have their transaction revert due to an out-of-bounds array access.


### Root Cause

In FeesManager.sol:169 the return variable newAmounts is declared as `uint256[] memory newAmounts` but is never initialized with new `uint256[](length)`. In Solidity, uninitialized dynamic memory arrays have length 0, so any write to newAmounts[i] at line 172 causes an out-of-bounds revert.

``` solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) // newAmounts has length 0
{
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // REVERTS: out-of-bounds access on zero-length array
    }
}
```

### Internal Pre-conditions

1. User needs to own a TVS NFT with at least 1 flow in the amounts array
2 splitFeeRate or mergeFeeRate can be any value (including 0)


### External Pre-conditions

None required.


### Attack Path

1. User calls `splitTVS()` with valid parameters to split their TVS position
2. `splitTVS()` calls calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows) at line 1069
3. Inside calculateFeeAndNewAmountForOneTVS, newAmounts is an uninitialized array with length 0
4. The loop attempts to write newAmounts[0] = amounts[0] - feeAmount
5. Transaction reverts with out-of-bounds array access error
6. Same flow occurs for mergeTVS() at line 1069

### Impact

All TVS holders cannot execute split or merge operations on their vesting positions. The splitTVS() and mergeTVS() functions are completely non-functional, representing a total denial of service for core protocol functionality. Users cannot manage their vesting positions as intended by the protocol design.

### PoC

```solidity
    function test_UninitializedArrayDoS() public {
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
    newAmounts = new uint256[](length); // Initialize array with proper length
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
        unchecked { ++i; }
    }
}
```
  