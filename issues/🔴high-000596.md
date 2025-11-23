# [000596] Users can claim/unhold all their TVS tokens and still receive dividend rewards due to a business logic error.
  
  ### Summary

In this protocol, there is a feature similar to a dividend contract that incentivizes users to hold TVSToken for as long as possible. The dividend is calculated based on the **unclaimedTVS**—the more unclaimed TVS, the higher the reward. According to the expected business logic, when a user claims all their TVS tokens, the `getUnclaimedAmount` function in the TVS contract should return 0.

However, a bug exists in the `claimTokens` function, specifically in this section of the code: [[AlignerzVesting.sol - Lines 941 to 975](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975). In this function, the `allocationOf` mapping is not updated, which is used to calculate the **unclaimableAmount** in the dividend distributor contract. The relevant code in the dividend distributor contract can be found here: [[A26ZDividendDistributor.sol - Lines 140 to 160](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L160)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L160).

As a result, users can claim all of their TVS tokens, but the `getUnclaimedAmount` function does not return 0 as expected. Instead, users are still able to claim dividends based on the full unclaimed amount. This breaks the expected business logic, as users can "unhold" their TVS tokens and continue receiving the full proportion of dividends.

### Root Cause

In the `claimTokens` function, found in this part of the code [[AlignerzVesting.sol - Lines 941 to 975](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975), the `allocationOf` mapping is not updated. This mapping is used to calculate the `getUnclaimableAmount` in the dividend distributor contract, leading to incorrect dividend calculations.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

1. There is a dividend contract designed to incentivize users to hold or not claim their TVS tokens for a certain period. The incentive is a dividend reward in USD.
2. The intended behavior is that when a user claims all of their TVS tokens, the dividend claim will return 0 since they have no unclaimed TVS remaining.
3. However, due to the `allocationOf` mapping not being updated, malicious users can still claim dividends even after they have claimed all of their TVS tokens.
4. This breaks the business logic, as users can unhold their tokens and continue receiving dividends.


### Impact

The business logic is broken, as users can unhold all their TVS tokens and still receive dividends.

### PoC


**"Add the `test_MissingAllocationOfUpdate` function to the test file, then run the following command in the terminal:**

```
forge test --mt test_MissingAllocationOfUpdate -vvv
```


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
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

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

    function test_MissingAllocationOfUpdate() public {
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

        vm.prank(projectCreator);
        A26ZDividendDistributor DividendDistributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp + 365 days, block.timestamp + 365 days + 365 days, address(token));
        
        deal(address(usdt), projectCreator, 15e18);

        vm.startPrank(projectCreator);
        usdt.transfer(address(DividendDistributor), 15e18);
        vm.stopPrank();
        
        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 365 days);

        for (uint256 i = 0; i < nftIds.length; i++) {
            console.log("unclaimed amount before claim all tvs token", DividendDistributor.getUnclaimedAmounts(nftIds[i]));
        }

        // 12. Users claim tokens after vesting period
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            //uint256 nftId = bidderNFTIds[bidder];

            uint256 tokenBalanceBefore = token.balanceOf(bidder);

            // console.log(tokenBalanceBefore);

            vm.prank(bidder);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);

            uint256 tokenBalanceAfter = token.balanceOf(bidder);

            // console.log(tokenBalanceAfter);
            assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
        }

        for (uint256 i = 0; i < nftIds.length; i++) {
            console.log("unclaimed amount after claim all tvs token", DividendDistributor.getUnclaimedAmounts(nftIds[i]));
        }

    }
}
```

The results below show that users can still receive dividends after claiming all their tokens.

```bash
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_MissingAllocationOfUpdate() (gas: 26618266)
Logs:
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount before claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000
  unclaimed amount after claim all tvs token 1000000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.09s (82.04ms CPU time)

Ran 1 test suite in 4.09s (4.09s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

do not forget to update allocationOf mapping state in the claimToken function
  