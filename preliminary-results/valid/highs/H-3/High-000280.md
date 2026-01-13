# [000280] `_computeSplitArrays(...)` Incorrectly Declares Allocation Struct Leading to `splitTVS(...)` to Be Unusable
  
  ### Summary

The `_computeSplitArrays(...)` function will always throw an `array out-of-bounds` error, leading to users not able to split their TVS tokens at all. That whole section of code is unusable. 

### Root Cause

If a user that has won a bid or reward (depending on the project they chose) they can claim an NFT, representing their TVS allocation. 

The contract also allows users who have multiple NFTs to merge them, and also if someone wants to split one NFT. 

However, there is a bug in the `_computeSplitArrays(...)` function, which will always revert. That function is an internal function allowing the calculations of splitting an NFT:

```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    ) internal view returns (Allocation memory alloc) { <@
        // ... //
    }
```

The marking above, shows that the `alloc` struct is declared there. 
Within the `Allocation` struct, there are arrays. The function tries to write to an uninitialized array. 

This line will throw straight away:
```solidity
alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

### Internal Pre-conditions

1. Owner has setup a bid/reward project.
2. Owner finalised bids.
3. User has one a bid/reward in TVS.
4. User want to split their TVS.

### External Pre-conditions

_No response_

### Attack Path

There is no Attack Path here. This is a protocol functionality not working.

### Impact

**Impact:** Medium <br>
The function is completely broken for its main use case (splitting into multiple pieces). Funds are not at risk. This is a protocol functionality not working at all. 

**Likelihood:** High <br>
Any user who tries to split their TVS will encounter this error. 

### PoC

Paste the following test in the `AlignerzVestingProtocolTest.t.sol` test suite:

```solidity
    function test_split_tvs() public {
        // Adding Attacker to the bidders list at the end index
        address attacker = makeAddr("attacker");
        vm.deal(attacker, 50 ether);
        bidders.push(attacker);
        usdt.mint(attacker, BIDDER_USD);

        // 1. Setting vesting divisor
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 2. Owner launches project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );

        // 2. Owner creates a single pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        // 3. Add bidders to whitelist
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // 4. Placing bids
        for (uint256 i = 0; i < bidders.length; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1)
                ? 180 days
                : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);

            vm.stopPrank();
        }

        // 5. Prepare bid allocations (this would be done off-chain)
        uint256 length = bidders.length;
        BidInfo[] memory allBids = new BidInfo[](length);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < length; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            bool accepted = i > 15; // last 15 bidders are accepted so attacker is accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1)
                    ? 180 days
                    : 365 days,
                poolId: 0,
                accepted: accepted
            });
        }

        // 6. Generate merkle root
        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoot, 60);

        // 9. Attacker claims NFT
        uint256[] memory perc = new uint256[](2);
        perc[0] = 5_000;
        perc[1] = 5_000;

        vm.startPrank(attacker);
        uint256 nftId = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[attacker]
        );

        vm.expectRevert(stdError.indexOOBError); // the array out of bounds error
        vesting.splitTVS(PROJECT_ID, perc, nftId);
        vm.stopPrank();
    }
```

### Mitigation

Consider modifying the `_computeSplitArrays()` function to the following:
```diff
function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    ) internal view returns (Allocation memory alloc) {
+       alloc.amounts = new uint256[](nbOfFlows);
+       alloc.vestingPeriods = new uint256[](nbOfFlows);
+       alloc.vestingStartTimes = new uint256[](nbOfFlows);
+       alloc.claimedSeconds = new uint256[](nbOfFlows);
+       alloc.claimedFlows = new bool[](nbOfFlows);
        // .Snip. //
    }
```
  