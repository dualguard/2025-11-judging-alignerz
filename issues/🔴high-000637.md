# [000637] [ High ]  Refund Mechanism Broken for Non-Pool 0 Bidders
  
  ### Summary

The hardcoded `poolId = 0` in the claimRefund function will cause a permanent loss of funds for all rejected bidders who were assigned to non-zero pools as the bidder will be unable to verify their Merkle proof against the refund root.

### Root Cause

In `AlignerzVesting.sol:claimRefund`, the Merkle leaf used for verification is constructed using a hardcoded `poolId = 0`.

```solidity
// AlignerzVesting.sol::claimRefund
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    // ...
    uint256 poolId = 0; // <--- The CRITICAL Hardcoded Flaw
    // ...
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof()); 
    // ...
}
```

The `refundRoot` must be constructed using the correct poolId for each rejected bidder to ensure proof integrity. Using `poolId = 0` guarantees the proof will fail for any user whose off-chain leaf was correctly generated with `poolId > 0.`

### Internal Pre-conditions

1. Admin needs to have called createPool to set up at least one pool with poolId > 0.

2. A user needs to have placed a bid that was ultimately rejected but was assigned to a non-zero pool ID (e.g., poolId = 1) during the allocation process.

3. The biddingProject.claimDeadline must not have passed (otherwise the funds are swept by the owner).

### External Pre-conditions

None

### Attack Path

- Project Setup: Admin calls createPool twice, creating Pool 0 and Pool 1.

- Bidding & Rejection: User Alice bids, and the off-chain system designates her rejected bid as belonging to` Pool 1`.

- Merkle Generation: The off-chain system correctly generates the refund leaf:` keccak256(Alice, amount, projectId, 1)`. This leaf is included in the final refundRoot.

- Action: Alice calls `vesting.claimRefund(...)` to recover her funds.

- Failure: The contract constructs a leaf using the hardcoded value: `keccak256(Alice, amount, projectId, 0)`.

- The contract executes require(MerkleProof.verify(...)), which reverts because the generated leaf `(poolId=0)` does not match the leaf in the root (poolId=1).

- Impact Trigger: Alice cannot recover her funds, and after the deadline, they become locked or recoverable only by the protocol owner via `withdrawPostDeadlineProfit`.

### Impact

The rejected bidders who were assigned to `non-Pool 0` tiers suffer a 100% loss of their bid `stablecoin (or equivalent of their bid amount)`. This loss is effectively transferred to the protocol's treasury when the administrator calls `withdrawPostDeadlineProfit` after the claim deadline passes

### PoC

Add this function to the `AlignerzVestingProtocolTest.t.sol`

```solidity

// ---
// --- PoC for Hardcoded PoolId=0 Refund Vulnerability ---
// ---

function test_olaoyesalem_HardcodedPoolId_RefundVulnerability() public {
    // SETUP: Create a project with multiple pools
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);
    
    // Launch project
    vesting.launchBiddingProject(
        address(token), 
        address(usdt), 
        block.timestamp, 
        block.timestamp + 1_000_000, 
        "0x0", 
        true
    );

    // Create multiple pools
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);  // Pool 0
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false); // Pool 1  
    vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false); // Pool 2

    // Whitelist bidders
    vesting.addUsersToWhitelist(bidders, PROJECT_ID);
    vm.stopPrank();

    // Place bids from different users
    for (uint256 i = 0; i < NUM_BIDDERS; i++) {
        vm.startPrank(bidders[i]);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;
        vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
        vm.stopPrank();
    }

    // Prepare bid allocations with users in different pools
    BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

    // Assign bidders to different pools
    for (uint256 i = 0; i < NUM_BIDDERS; i++) {
        uint256 poolId = i % 3; // Distribute across pools 0, 1, 2
        bool accepted = i < 10; // Only first 10 bidders accepted
        
        allBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD,
            vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
            poolId: poolId,
            accepted: accepted
        });
    }

    // Generate merkle roots for each pool
    bytes32[] memory poolRoots = new bytes32[](3);
    for (uint256 poolId = 0; poolId < 3; poolId++) {
        poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
    }

    // Generate refund proofs - NOTE: This uses the CORRECT poolIds for each user
    refundRoot = generateRefundProofs(allBids);

    // Finalize project
    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

    // TEST: Accepted bidders claim NFTs successfully
    for (uint256 i = 0; i < 10; i++) {
        address bidder = bidders[i];
        uint256 poolId = bidderPoolIds[bidder];
        
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
        assertEq(nft.ownerOf(nftId), bidder);
        bidderNFTIds[bidder] = nftId;
    }

    // CRITICAL TEST: Rejected bidders try to claim refunds
    for (uint256 i = 10; i < NUM_BIDDERS; i++) {
        address bidder = bidders[i];
        uint256 correctPoolId = bidderPoolIds[bidder]; // This is their actual pool ID (0, 1, or 2)
        bytes32[] memory proof = bidderProofs[bidder];
        
        console.log("Testing refund for bidder:", bidder);
        console.log("Correct poolId:", correctPoolId);
        console.log("USDT balance before:", usdt.balanceOf(bidder));

        vm.prank(bidder);
        
        // This will FAIL for users in pools 1 and 2 because claimRefund hardcodes poolId=0
        if (correctPoolId == 0) {
            // Pool 0 users can claim refunds successfully
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, proof);
            assertEq(usdt.balanceOf(bidder), BIDDER_USD, "Pool 0 user should get refund");
            console.log("Pool 0 user SUCCESSFULLY claimed refund");
        } else {
            // Pool 1 and 2 users CANNOT claim refunds due to hardcoded poolId=0
            vm.expectRevert(); // Merkle proof verification fails
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, proof);
            
            // Verify funds are still locked
            assertEq(usdt.balanceOf(bidder), 0, "Non-pool 0 user should NOT get refund");
            console.log("Pool", correctPoolId, "user FAILED to claim refund - FUNDS LOCKED");
        }
    }

    // VERIFICATION: Calculate total locked funds
    uint256 totalLockedFunds = 0;
    for (uint256 i = 10; i < NUM_BIDDERS; i++) {
        address bidder = bidders[i];
        uint256 poolId = bidderPoolIds[bidder];
        if (poolId != 0) {
            totalLockedFunds += BIDDER_USD;
        }
    }
    
    console.log("Total locked funds due to hardcoded poolId=0:", totalLockedFunds);
    console.log("Affected users:", (NUM_BIDDERS - 10) * 2 / 3); // ~2/3 of rejected bidders are in pools 1-2
}

```

### Mitigation

- Fix the `claimRefund` function:
```solidity
function claimRefund(
    uint256 projectId,
    uint256 amount,
    uint256 poolId, // <--- ADD THIS ARGUMENT
    bytes32[] calldata merkleProof
) external {
    // ...
    // Use the argument instead of the hardcoded value:
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId)); 
    // ...
}
```
  