# [000966] The user may lose all their TVS and unallocated tokens due to an error in the merger.
  
  ### Summary

The `mergeTVS` function does not check whether `mergedNftId` is in the `nftIds` array. If the user accidentally passes the same NFT ID to both parameters, the NFT will be burned and the user will lose it forever. 

### Root Cause

There is no input validation to prevent `mergedNftId` from being passed to the `nftIds` array.

The `mergeTVS` function only checks:
- NFT ownership
- Non-empty nftIds array
- Array length matching 

### Internal Pre-conditions
Change this function so that the `mergeTVS` function can be called.

```solidity 
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        uint256[] memory newAmounts = new uint256[](length);
        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); 
            newAmounts[i] = amounts[i] - feeAmount; 
        }
        return (feeAmount, newAmounts);
    }
```

### External Pre-conditions

_No response_

### Attack Path

1. The user has several TVS that they want to merge.
2. They call the mergeTVS function and mistakenly pass the main token that was supposed to be merged to the array for merging. 

### Impact

The user loses all their funds, NFTs, tokens, without any chance of recovery. 

### PoC

```solidity 
  function test_burnNFT() public {
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
        nftIds[0] = nftId0;

        assertEq(nft.ownerOf(nftId0), bidder0);
        uint256 mergedNftId = vesting.mergeTVS(
            PROJECT_ID,
            nftId0,
            projectIds,
            nftIds
        );
        vm.expectRevert(
            abi.encodeWithSignature("OwnerQueryForNonexistentToken()")
        );
        nft.ownerOf(nftId0);
    }
```

### Mitigation

Add a check to the loop to ensure that the main token is not equal to the token that will be burned. 

```diff
 for (uint256 i; i < nbOfNFTs; i++) {
+           require(nftIds[i] != mergedNftId);
            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token); //@audit почему взымают два раза комиссию ?
        }
```
  