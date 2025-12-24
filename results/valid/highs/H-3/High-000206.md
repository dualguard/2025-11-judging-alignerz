# [000206] Uninitialized Dynamic Arrays in _computeSplitArrays Causes Complete DoS of Split Functionality
  
  ### Summary

In `AlignerzVesting.sol:1113`, the returned Allocation memory alloc struct contains dynamic arrays that are never initialized, which will cause a complete denial of service for all TVS holders as any user calling splitTVS() will have their transaction revert due to out-of-bounds array access when writing to alloc.amounts[j].

### Root Cause

In AlignerzVesting.sol:1113, the function returns an Allocation memory alloc struct but never initializes its dynamic array members (amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows) before writing to them:

``` solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (Allocation memory alloc) // alloc.amounts has length 0
{
    // ... base arrays loaded correctly ...
    
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // REVERTS: length is 0
        alloc.vestingPeriods[j] = baseVestings[j];    // Would also revert
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j]; // Would also revert
        alloc.claimedSeconds[j] = baseClaimed[j];     // Would also revert
        alloc.claimedFlows[j] = baseClaimedFlows[j];  // Would also revert
        unchecked { ++j; }
    }
}

```

### Internal Pre-conditions

1. User needs to own a TVS NFT with at least 1 flow in the allocation
2. User needs to call splitTVS() with valid percentages array

### External Pre-conditions

None required.

### Attack Path

1. User calls `splitTVS()` with valid parameters (e.g., percentages = [5000, 5000] for 50/50 split)
2. `splitTVS()` passes fee calculation (assuming that bug is fixed)
3. Loop at line 1082 calls `_computeSplitArrays(allocation, percentage, nbOfFlows)` at line 1113
4. Inside _computeSplitArrays, alloc is a memory struct with all dynamic arrays having length 0
5. First write attempt alloc.amounts[0] = ... at line 1132 causes out-of-bounds revert
6.  Transaction reverts, user cannot split their TVS

### Impact

All TVS holders cannot execute split operations on their vesting positions. The `splitTVS()` function is completely non-functional, representing a total denial of service for core protocol functionality. Users cannot partially exit positions, cannot transfer partial vesting rights, and cannot manage their TVS portfolios as intended by the protocol design.


### PoC

``` solidity

    function test_ComputeSplitArraysDoS() public {
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
...
```

### Mitigation

_No response_
  