# [000182] `BidUpdated` event can be emitted even though user didn't update the bid.
  
  ### Summary

The `updateBid` function allows a user to update his bid, as long as `newAmount >= oldAmount` and `newVestingPeriod >= bid.vestingPeriod`.  These checks ensure that either the amount or the vesting period cannot be lower than they were in the previous bid.

The problem is that there is no check to ensure the bid is not identical, allowing to call `updateBid` without actually updating anything.

### Root Cause

Missing check to ensure it is not possible to update the bid with an identical bid.

### Attack Path

Any user who called `placeBid` can update the bid with the same bid, emitting the `BidUpdated` event while nothing was updated.

### Impact

The impact of this issue is low as it results in an incorrect event emission which might be misleading for off-chain monitoring.

### PoC

Please copy paste the following test in the test file:

```solidity
  function testUpdateBidWithSameBid() external {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", false
        );

        // 2. Create pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        vm.stopPrank();

        // 3. Place bids
        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD * 2); // Approve for 2 bids
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 90 days); // @audit LOW same bid is possible
        vm.stopPrank();
    }
```

This test shows that it possible to place a bid, and then update this bid with the same amount and same vesting period.
When we look at the trace, we see that the `BidUpdated` event is emitted with the same values:

```solidity
    │   │   ├─ emit BidUpdated(projectId: 0, user: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], oldAmount: 1000000000000000000000 [1e21], newAmount: 1000000000000000000000 [1e21], oldVestingPeriod: 7776000 [7.776e6], newVestingPeriod: 7776000 [7.776e6])
```

### Mitigation

Ensure that `updateBid` function has a check to prevent identical bidding:

```solidity
        require(newAmount > oldAmount || newVestingPeriod > oldVestingPeriod, "Same bid forbidden");
```


  