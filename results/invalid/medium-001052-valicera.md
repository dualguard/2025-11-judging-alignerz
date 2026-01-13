# [001052] placeBid will revert if bidFee is not accounted into amount
  
  ### Summary

`AlignerzVesting::placeBid` does two separate pulls - it transfers amount to the contract and separately transfers bidFee to the treasury if set. So the bidder must have and approve both amount + fee. If they only approve/hold amount bid will revert.

### Root Cause

`AlignerzVesting::placeBid` does two separate transfers:
```solidity
 function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        // ...
        //@audit bidfee is not accounted to amount param, so if user didnt provide amount + fee
        // it will revert due to insufficient balance
>>   biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);
        if (bidFee > 0) {
>>        biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, bidFee);
        }
        biddingProject.bids[msg.sender] = Bid({amount: amount, vestingPeriod: vestingPeriod});
        biddingProject.totalStablecoinBalance += amount;

        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
    }
```
Those are two independent ERC20 transferFrom calls that each require sufficient balance and sufficient allowance. If the caller only approved or holds exactly amount but not amount + bidFee, the second call will revert with an ERC20 transfer/allowance error and the whole transaction fails — even though the intent was to place the bid for amount. In practice the user must have balance >= amount+bidFee and allowance >= amount+bidFee (or two approvals covering each call).

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L708-L735

### Internal Pre-conditions

1. bidFee must be set

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Users bids will revert if fee is not accounted into amount. Likelihood is high, Impact is low since funds are refunded with revert.

### PoC

Add test in `AlignerzVestingProtocolTest` and run `forge test --mt test_placeBid_requires_approve_amountPlusFee -vvvv`

```solidity
    function test_placeBid_requires_approve_amountPlusFee() public {
        uint256 amount = BIDDER_USD;
        // set bidFee to a small absolute value within the current FeesManager cap (< 100001)
        // FeesManager currently treats bidFee as an absolute token amount (not bps)
        uint256 fee = 100000;

        // projectCreator is owner of vesting (set in setUp)
        vm.prank(projectCreator);
        vesting.setBidFee(fee);

        // Launch bidding project and create a pool with whitelist enabled
        vm.prank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );
        vm.prank(projectCreator);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        // whitelist bidder[0]
        address bidder = bidders[0];
        address[] memory wl = new address[](1);
        wl[0] = bidder;
        vm.prank(projectCreator);
        vesting.addUsersToWhitelist(wl, PROJECT_ID);

        // Case A - bidder approves ONLY the amount (not amount+fee) -> placeBid should revert
        vm.prank(bidder);
        usdt.approve(address(vesting), amount);

        vm.startPrank(bidder);
        vm.expectRevert();
        vesting.placeBid(PROJECT_ID, amount, 90 days);
        vm.stopPrank();

        // Give the bidder the extra fee amount so they can cover amount+fee
        usdt.mint(bidder, fee);

        // Case B - bidder approves amount+fee and placeBid succeeds
        vm.prank(bidder);
        usdt.approve(address(vesting), amount + fee);

        vm.prank(bidder);
        vesting.placeBid(PROJECT_ID, amount, 90 days);

        // After successful bid: contract should hold `amount`, treasury should have `fee`, bidder balance is zero
        assertEq(usdt.balanceOf(address(vesting)), amount);
        assertEq(usdt.balanceOf(address(1)), fee);
        assertEq(usdt.balanceOf(bidder), 0);
    }
```

### Mitigation

Pull a single amount (amount + fee) in one safeTransferFrom, then forward the fee to treasury.
  