# [000971] Incorrect implementation of `allocationOf` mapping leads to a persistent DoS in all functions that call this mapping.
  
  ### Summary

The `AlignerzVesting` contract declares `allocationOf` as a public mapping of struct `Allocation` containing dynamic arrays. Solidity's auto-generated getter for such mappings only returns static fields and cannot return dynamic arrays. This causes all external contracts relying on `allocationOf()` — particularly `A26ZDividendDistributor.getUnclaimedAmounts()` — to revert when trying to access array fields, resulting in permanent DoS 

### Root Cause

The core issue stems from Solidity's limitation in auto-generating getters for public mappings:

When a public mapping points to a struct with dynamic arrays, the compiler generates a getter that returns only non-array fields:

This means calls like `vesting.allocationOf(nftId).amounts` result - revert 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user receives a TVS token.
2. The user calls getUnclaimedAmounts.

### Impact

- All NFTs minted, merged, or split cannot have their dividend allocations read by `A26ZDividendDistributor`
- Users permanently lose access to dividend distribution for these NFTs
- Protocol's core revenue-sharing mechanism becomes non-functional

### PoC

```solidity
   function test_AllocationOfDoS() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false
        );

        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            usdt.approve(address(vesting), BIDDER_USD);

            uint256 vestingPeriod = (i % 3 == 0)
                ? 90 days
                : (i % 3 == 1)
                    ? 180 days
                    : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            uint256 poolId = i % 3;
            bool accepted = i < 15;

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD,
                vestingPeriod: (i % 3 == 0)
                    ? 90 days
                    : (i % 3 == 1)
                        ? 180 days
                        : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        refundRoot = generateRefundProofs(allBids);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

        address bidder0 = bidders[0];
        address bidder1 = bidders[1];

        vm.prank(bidder0);
        uint256 nftId0 = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[bidder0]
        );

        vm.prank(bidder1);
        uint256 nftId1 = vesting.claimNFT(
            PROJECT_ID,
            1, // poolId 1
            BIDDER_USD,
            bidderProofs[bidder1]
        );

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

        dividendDistributor.getUnclaimedAmounts(mergedNftId);
    }
```

### Mitigation

Rename the internal mapping and implement an explicit view function that returns the complete Allocation struct

```solidity 
mapping(uint256 => Allocation) internal _allocationOf;

function allocationOf(uint256 nftId) external view returns (Allocation memory) {
    return _allocationOf[nftId];
}
```
  