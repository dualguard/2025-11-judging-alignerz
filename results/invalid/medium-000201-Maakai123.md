# [000201] [Medium]    Attackers will strand refunds/NFTs by expiring the single claim window
  
  ### Summary

 The lone `claimDeadline` set during `finalizeBids` (protocol/src/contracts/vesting/AlignerzVesting.sol:783-805) gates both `claimRefund` and `claimNFT` (protocol/src/contracts/vesting/AlignerzVesting.sol:835-890), so once `block.timestamp >= claimDeadline` every Merkle proof becomes unusable even when the user never transacted. The leftover `totalStablecoinBalance` is then swept to treasury via `withdrawPostDeadlineProfit` (protocol/src/contracts/vesting/AlignerzVesting.sol:914-934), so anyone who misses the window—whether from gas spikes  permanently loses their refund/allocation. Although the comment on `claimDeadline` says late claims are “impossible”, this design choice is flawed because it turns merkle commitments into a race instead of a guaranteed right and exposes users to griefing attacks that the protocol cannot undo.

Side note


Even though the struct comment explicitly says “claiming TVS or refund is impossible after claimDeadline,” that design choice breaks the stronger guarantee implied by the rest of the system: a bidder with a valid Merkle proof should always be able to redeem what they fairly earned. In practice, a sale sets that deadline in finalizeBids, and the contract never offers a late-claim, owner-assisted recovery, or per-claim grace period. If the chain congests or the backend/front-end fails during the claim window—events outside the user’s control—every honest claimant is permanently locked out, yet the protocol treats their money as “profit” and sweeps it to treasury. So it’s a deliberate behavior, but it’s still a bug because the intent (fair Merkle-backed allocations) and the expectation communicated to users (“prove your claim”) are violated whenever there’s a transient outage. 


### Root Cause

In protocol/src/contracts/vesting/AlignerzVesting.sol:803-890 every bidder-facing claim path enforces `require(claimDeadline > block.timestamp)` while protocol/src/contracts/vesting/AlignerzVesting.sol:914-934 treats any post-deadline balance as profit. No late-claim, owner override, or per-claim grace period exists, so the contract intentionally conflates “not yet claimed” with “forfeit to treasury”, making liveness entirely dependent on a single window.

### Internal Pre-conditions

 1. Owner finalizes the sale and sets `claimDeadline = block.timestamp + claimWindow` (protocol/src/contracts/vesting/AlignerzVesting.sol:801-804).
2. Refund and allocation merkle roots are published, creating provable but unclaimed entitlements.
3. A subset of bidders has not yet called `claimRefund`/`claimNFT` before the deadline fires.


### External Pre-conditions

 External Pre-conditions
1. Blockspace congestion, or MEV spam  prevents some users from getting transactions mined before `claimDeadline` (no admin misconfiguration or user error is required; a single network event suffices).

### Attack Path

Attack Path
1. Sale finalizes and attackers learn the exact `claimDeadline`.
2. During the claim window they spam the network (or simply wait for organic congestion/front-end issues) so that small users’ gas limits are never met.
3. Deadline expires; all `claimRefund`/`claimNFT` calls revert with `Deadline_Has_Passed`.
4. Treasury (or any owner-controlled role) calls `withdrawPostDeadlineProfit` to confiscate `totalStablecoinBalance`, which now includes every stranded refund and unminted NFT allocation.

### Impact

 Honest bidders permanently lose their refunds or NFTs, while treasury ends up with the seized balance. Attackers gain griefing leverage despite not profiting directly. Likelihood: Medium — a congested few hours or chain halt during any sale is realistic and there is zero remediation once the deadline passes. Dependency on admin misconfiguration or caller misuse: No — the protocol behaves this way even with reasonable `claimWindow` inputs; the vulnerability stems from the lack of any post-deadline recovery path, so an external DoS or unforeseen outage is enough to trigger it.

### PoC

````solidity
 function test_BiddingClaimsExpireAndFundsAreSwept() public {
        uint256 projectId = 0;
        uint256 poolId = 0;
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        uint256 startTime = block.timestamp;
        vesting.launchBiddingProject(address(token), address(usdt), startTime, startTime + 1_000, "0x0", true);
        vesting.createPool(projectId, 1_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, projectId);
        vm.stopPrank();

        for (uint256 i; i < 2; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(projectId, BIDDER_USD, 30 days);
            vm.stopPrank();
        }

        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({bidder: bidders[0], amount: BIDDER_USD, vestingPeriod: 30 days, poolId: poolId, accepted: true});
        bids[1] = BidInfo({bidder: bidders[1], amount: BIDDER_USD, vestingPeriod: 30 days, poolId: poolId, accepted: false});

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(bids, poolId);
        refundRoot = generateRefundProofs(bids);

        uint256 finalizeTime = block.timestamp;
        vm.prank(projectCreator);
        vesting.finalizeBids(projectId, refundRoot, poolRoots, 2);
        vm.warp(finalizeTime + 3);

        vm.startPrank(bidders[0]);
        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vesting.claimNFT(projectId, poolId, BIDDER_USD, bidderProofs[bidders[0]]);
        vm.stopPrank();

        vm.startPrank(bidders[1]);
        vm.expectRevert(AlignerzVesting.Deadline_Has_Passed.selector);
        vesting.claimRefund(projectId, BIDDER_USD, bidderProofs[bidders[1]]);
        vm.stopPrank();

        uint256 contractBalance = usdt.balanceOf(address(vesting));
        uint256 treasuryBefore = usdt.balanceOf(address(1));
        vm.prank(projectCreator);
        vesting.withdrawPostDeadlineProfit(projectId);
        assertEq(usdt.balanceOf(address(vesting)), 0, "Unclaimed funds remain in contract");
        assertEq(usdt.balanceOf(address(1)), treasuryBefore + contractBalance, "Treasury failed to sweep stranded funds");
    }
```

### Mitigation

Keep claimed funds separate from extra profit. Instead of sweeping totalStablecoinBalance, track unclaimed refunds vs. actual surplus and only allow treasury withdrawals of the true profit, leaving the merkle-committed portion retrievable.
  