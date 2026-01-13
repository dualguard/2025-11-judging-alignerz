# [001045] Blacklisted user can still update his bid
  
  ### Summary

Current bidding behaviour allows a whitelisted bidder to place a bid, the owner can later remove that bidder from the whitelist (blacklist them), and the bidder can still call updateBid(...) successfully.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L776

### Root Cause

`AlignerzVesting::placeBid` enforces whitelist status when whitelisting is enabled,
it checks if (isWhitelistEnabled[projectId]) require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
```solidity
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
>>          require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
         // ...
    }
```
`updateBid` does not check isWhitelisted at all. It only validates project/time/amount/vesting and then accepts additional stablecoin from msg.sender.

```solidity
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        //@audit user who is blacklisted after placing bid can still update bid
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime
                && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );

        Bid storage bid = biddingProject.bids[msg.sender];
        uint256 oldAmount = bid.amount;
        require(oldAmount > 0, No_Bid_Found());
        require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());
        require(newVestingPeriod > 0, Zero_Value());
        require(newVestingPeriod >= bid.vestingPeriod, New_Vesting_Period_Cannot_Be_Smaller());
        // ...
    }
```
Therefore removing a user from the whitelist after they placed a bid does not prevent them from calling updateBid because no whitelist check exists in `updateBid`.

### Internal Pre-conditions

1. Bidding project has to be launched.
2. User have to be whitelisted and must have placed bid.
3. Owner must have blacklisted him after placed bid.

### External Pre-conditions

N/A

### Attack Path

1. Attacker places a bid as whitelisted user.
2. Owner blacklists him before ending period.
3. Attacker updates his bid before ending essentially gaming the bid.

### Impact

Owners cannot prevent previously-whitelisted bidders from modifying their bids after removal from the whitelist. This breaks the intended control that removeUsersFromWhitelist should provide during an active bidding window.
It allows bypassing of intended access control during active bidding windows

### PoC

In `AlignerzVestingProtocolTest` paste the following test and run `forge test --mt test_BlacklistedUserCanStillUpdateBid -vvvv`

```solidity
    function test_BlacklistedUserCanStillUpdateBid() public {
        uint256 amount = BIDDER_USD;
        address bidder = bidders[0];

        // projectCreator is owner of vesting (set in setUp)
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);

        // whitelist bidder
        address[] memory wl = new address[](1);
        wl[0] = bidder;
        vesting.addUsersToWhitelist(wl, PROJECT_ID);
        vm.stopPrank();

        // bidder places a bid
        vm.prank(bidder);
        usdt.approve(address(vesting), amount);
        vm.prank(bidder);
        vesting.placeBid(PROJECT_ID, amount, 90 days);

        // Owner removes bidder from whitelist (blacklist)
        address[] memory rem = new address[](1);
        rem[0] = bidder;
        vm.prank(projectCreator);
        vesting.removeUsersFromWhitelist(rem, PROJECT_ID);

        // Bidder attempts to update bid after being blacklisted
        // Give a tiny extra amount so updateBid would transfer additional funds if it runs
        usdt.mint(bidder, 1);
        vm.prank(bidder);
        usdt.approve(address(vesting), amount + 1);

        // updateBid still succeeds even after removal from whitelist
        vm.prank(bidder);
        vesting.updateBid(PROJECT_ID, amount + 1, 180 days);

        // Assert the vesting contract received the extra amount -> update executed
        assertEq(usdt.balanceOf(address(vesting)), amount + 1);
    }
```

### Mitigation

Mirror the whitelist guard from placeBid inside updateBid so update operations are blocked for removed (blacklisted) users while whitelist enforcement is active.
```solidity
if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
}
```
  