# [000262] Refund Logic Trusts Off-Chain Merkle Construction, Allowing Incorrect or Excessive Refund Amounts
  
  ### Summary

claimRefund() does not verify that the refund amount is consistent with the bidder’s actual on-chain bid. Instead, the function trusts the amount embedded in the Merkle leaf. Because the contract performs no upper-bound enforcement, any mistake or manipulation in the off-chain Merkle construction process can cause a bidder to receive more than their true bid amount, or receive refunds in situations that should not be eligible.

Merkle proofs protect integrity, not correctness. If the Merkle tree is built incorrectly off-chain—or encodes wrong amounts—the smart contract will still accept it as valid.

### Root Cause

claimRefund accepts a user-supplied amount and verifies it only through Merkle inclusion.
The contract does not check that this amount matches or is bounded by the bidder’s actual on-chain bid.amount.

This means the correctness of refund amounts is not enforced on-chain.
It is entirely delegated to the off-chain Merkle tree generation script.

This breaks a critical invariant:

Refund amounts must correspond to the actual bid amount recorded on-chain.

They do not check if amount is correct, amount ≤ bid.amountthe, leaf corresponds to a rejected bid.

```solidity
    function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());


        Bid storage bid = biddingProject.bids[msg.sender];
        require(bid.amount > 0, No_Bid_Found());


        uint256 poolId = 0;
        // Verify merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
        require(!claimedRefund[leaf], Already_Claimed());
        require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
        claimedRefund[leaf] = true;


        biddingProject.totalStablecoinBalance -= amount;
        biddingProject.stablecoin.safeTransfer(msg.sender, amount);


        emit BidRefunded(projectId, msg.sender, amount);
    }
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- Attacker makes a bid of 100.

- Off-chain script (buggy, manipulated, or misindexed) writes a leaf encoding amount = 2000.

- Attacker claims:

```solidity
claimRefund(projectId, 2000, proof)
```

- The Merkle proof is valid because the incorrect leaf was included in the tree.

- Contract transfers 2000 because the contract has no internal guardrail to detect the incorrect amount.

### Impact

- Because the refund amounts are not validated against on-chain state, the system can enter inconsistent states:

- transferred stablecoins do not match bid records,

- totalStablecoinBalance becomes inaccurate,

- subsequent accounting (e.g., distribution, treasury operations) becomes unreliable.

### PoC

Please add this to the

```solidity
    function test_StealViaClaimRefund() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

        // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        uint256[] memory nftIds = new uint256[](15);
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders .......this concept is lol. you whitelisted b4 bid and then don't accept
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            vm.prank(bidder);
            //(uint256 oldNft, uint256 newNft) = vesting.splitTVS(PROJECT_ID, nftId, 5000);
            //assertEq(oldNft, nftId);
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 10. Some users try to claim refunds
        for (uint256 i = 1; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.expectRevert();
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD + 1, bidderProofs[bidder]);
//Here the attacker was able to steal more than they bidded with refund.

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }


    }
```

### Mitigation

Enforce on-chain upper bound (minimum fix)

```require(amount <= bid.amount, ExcessiveRefund());
```

Enforce exact refund amount (preferred)

```
require(amount == bid.amount, InvalidRefundAmount());
```
  