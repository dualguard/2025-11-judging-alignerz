# [000628] User will encounter array out-of-bounds revert when attempting to merge or split NFTs due to uninitialized `newAmounts` array
  
  ### Summary

Uninitialized memory array `newAmounts` in `calculateFeeAndNewAmountForOneTVS` function will cause array out-of-bounds access for any user attempting to merge or split NFTs, as the function attempts to write to an array with zero length.

### Root Cause

in [AlignerzVesting.sol:calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C5-L174C6) the `newAmounts` array is declared but never initialized with a length:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // newAmounts is declared but never initialized
    // newAmounts.length = 0
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount; //  Accessing index i of zero-length array
        }
    }
```
When `newAmounts[i] = ...` executes, it attempts to write to index i of an array with length 0, causing an out-of-bounds revert (panic code 0x32).

### Internal Pre-conditions

User must call either `mergeTVS `or `splitTVS` functions
NFTs must have at least 1 flow (amounts.length >= 1)

### External Pre-conditions

_No response_

### Attack Path

1. user calls splitTVS function
2. function calls `calculateFeeAndNewAmountForOneTVS`
3. Inside function: `newAmounts` has length 0
4. Solidity checks: 0 < 0.length → 0 < 0 → panic: array out-of-bounds (0x32)

### Impact

Users cannot merge or split NFTs, breaking two core protocol features

### PoC

add this function in `AlignerzVestingProtocolTest.t.sol` file
```solidity
function test_MergeTVS_Bug() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vesting.launchBiddingProject(
            address(token), 
            address(usdt), 
            block.timestamp, 
            block.timestamp + 1_000_000, 
            "0x0", 
            true
        );

        // 2. Create pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, false);

        // 3. Whitelist bidders
        address[] memory selectedBidders = new address[](3);
        selectedBidders[0] = bidders[0];
        selectedBidders[1] = bidders[1];
        selectedBidders[2] = bidders[2];
        vesting.addUsersToWhitelist(selectedBidders, PROJECT_ID);
        vm.stopPrank();

        // 4. Three users place bids
        for (uint256 i = 0; i < 3; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
            vm.stopPrank();
        }

        // 5. Prepare allocations - all 3 bidders accepted
        BidInfo[] memory allBids = new BidInfo[](3);
        for (uint256 i = 0; i < 3; i++) {
            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD,
                vestingPeriod: 90 days,
                poolId: 0,
                accepted: true
            });
        }

        // 6. Generate merkle roots
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        // 7. Finalize project
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 8. All three users claim NFTs
        uint256[] memory nftIds = new uint256[](3);
        for (uint256 i = 0; i < 3; i++) {
            vm.prank(bidders[i]);
            nftIds[i] = vesting.claimNFT(
                PROJECT_ID, 
                0, 
                BIDDER_USD, 
                bidderProofs[bidders[i]]
            );
            
            // Verify NFT ownership
            assertEq(nft.ownerOf(nftIds[i]), bidders[i]);
        }

        // 9. First user tries to merge all 3 NFTs
        // First, transfer NFTs from bidders[1] and bidders[2] to bidders[0]
        vm.prank(bidders[1]);
        nft.transferFrom(bidders[1], bidders[0], nftIds[1]);
        
        vm.prank(bidders[2]);
        nft.transferFrom(bidders[2], bidders[0], nftIds[2]);

        // Now bidders[0] owns all 3 NFTs
        assertEq(nft.ownerOf(nftIds[0]), bidders[0]);
        assertEq(nft.ownerOf(nftIds[1]), bidders[0]);
        assertEq(nft.ownerOf(nftIds[2]), bidders[0]);

        // 10. Try to merge NFTs - THIS SHOULD REVERT due to infinite loop bug
        uint256[] memory nftsToMerge = new uint256[](2);
        nftsToMerge[0] = nftIds[1];
        nftsToMerge[1] = nftIds[2];

        uint256[] memory projectIds = new uint256[](2);
        projectIds[0] = PROJECT_ID;
        projectIds[1] = PROJECT_ID;

        vm.prank(bidders[0]);
        
        // This will REVERT with an out-of-gas error due to infinite loop
        // in calculateFeeAndNewAmountForOneTVS
        //vm.expectRevert(); // Expecting revert due to infinite loop/out of gas
        vesting.mergeTVS(
            PROJECT_ID, 
            nftIds[0],  // Keep this NFT
            projectIds, 
            nftsToMerge // Merge these into nftIds[0]
        );
    }
```

Run 
```js
forge test --mt test_MergeTVS_Bug
```
the output:
```
[FAIL: panic: array out-of-bounds access (0x32)] test_MergeTVS_Bug() (gas: 3747766)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.69s (7.52ms CPU time)
```

### Mitigation

Initialize the `newAmounts` array 
```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+     newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }

```
  