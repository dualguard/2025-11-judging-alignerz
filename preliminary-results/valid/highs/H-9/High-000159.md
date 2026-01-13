# [000159] Dividend distribution is DoSed due to invalid struct readings
  
  ### Summary

External calls to mapping structs will only return their static parameters, not the dynamic ones, this makes the dividend system not work.

### Root Cause

In the getUnclaimedAmounts function:
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
allocationOf is a public mapping in the vesting contract, so this call suceeds. However, because of the way solidity works, calls to this mapping only return static elements, not dynamic ones. As a result, it only returns isClaimed, token, assignedPoolId.

When attempting to read .token, the call reverts because this element does not exist in this context.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

-

### Impact

All calls to getUnclaimedAmounts will fail, the worst consequence is that the entire dividend system will never work. No funds are lost due to admin retrieval functions however.

### PoC

```solidity
function test_PoC_InvalidCall() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        address user1 = bidders[0];
        address user2 = bidders[1];
        address[] memory twoBidders = new address[](2); // Corrected from `new address[]`
        twoBidders[0] = user1;
        twoBidders[1] = user2;
        vesting.addUsersToWhitelist(twoBidders, PROJECT_ID);
        vm.stopPrank();

        // --- 2. Users place bids ---
        vm.startPrank(user1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(user2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({bidder: user1, amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true});
        allBids[1] = BidInfo({bidder: user2, amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // --- 4. Users claim NFTs, triggering the storage corruption ---
        // User 1 claims, `allocationOf[nftId1]` gets a storage pointer.
        vm.prank(user1);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[user1]);

        // User 2 claims. This reuses the same storage slot for `biddingProject.allocations`,
        // corrupting the data `allocationOf[nftId1]` was pointing to.
        vm.prank(user2);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[user2]);

        vm.prank(owner);
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdt), block.timestamp, 90 days, address(token)
        );
        usdt.mint(address(distributor), 1_000_000 * 1e6); // Fund distributor

        //create a different token to bypass the early return in getUnclaimedAmounts
        Aligners26 token2 = new Aligners26("Another Token", "ATK");
        distributor.setToken(address(token2));
        
        //this will revert because vesting.allocationOf(1) does not return dynamic arrays
        vm.expectRevert(bytes(""));
        distributor.getUnclaimedAmounts(1);
    }
```

### Mitigation

You need to add a helper function in the vesting contract in order to return dynamic arrays.
  