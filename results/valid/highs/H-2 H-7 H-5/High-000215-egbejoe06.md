# [000215] Merge and Split TVS Operations Broken Due to Critical Bugs in Fee Calculation
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function contains three critical bugs that cause complete denial of service for merge and split operations: (1) the `newAmounts` array is never allocated before use, (2) the loop variable `i` is never incremented causing an infinite loop, and (3) the fee calculation logic incorrectly subtracts cumulative fees from each individual amount. These bugs cause all merge and split operations to revert with panic errors.

### Root Cause

## Root Cause

In `protocol/src/contracts/vesting/feesManager/FeesManager.sol:169-174`, the `calculateFeeAndNewAmountForOneTVS` function contains three critical bugs:

### Bug #1: Unallocated Array
The `newAmounts` array is declared as a return value but never allocated with `new uint256[](length)` before attempting to write to it. This causes a panic revert with error code `0x32` (array out-of-bounds access) on the first iteration.

### Bug #2: Infinite Loop
The loop uses `for (uint256 i; i < length;)` without incrementing `i`. Even if Bug #1 were fixed, this would cause an infinite loop that exhausts gas.

### Bug #3: Incorrect Fee Calculation
The function subtracts the cumulative `feeAmount` from each individual amount instead of subtracting only the fee for that specific amount. This causes later amounts in the array to be over-charged.

### Internal Pre-conditions

1. User needs to own a TVS NFT (obtained through `claimNFT` after placing a bid and project finalization)
2. For merge: User needs to own multiple TVS NFTs to merge, or a TVS with multiple flows
3. For split: User needs to call `splitTVS` function with valid parameters (projectId, percentages array, NFT ID)
4. The TVS must have at least one flow (which is always the case for valid TVS NFTs)

### External Pre-conditions

_No response_

### Attack Path

## Attack Path

### For Merge Operations:
1. User places bids in a bidding project and receives TVS NFTs through `claimNFT`
2. User attempts to merge TVS NFTs by calling `mergeTVS(projectId, mergedNftId, projectIds, nftIds)`
3. The `mergeTVS` function calls `calculateFeeAndNewAmountForOneTVS` on line 1013 to calculate fees for the merged TVS flows
4. `calculateFeeAndNewAmountForOneTVS` attempts to write to `newAmounts[0]` without first allocating the array
5. Solidity panics with error code `0x32` (array out-of-bounds access)
6. The entire transaction reverts, preventing the user from merging their TVS NFTs

### For Split Operations:
1. User places a bid in a bidding project and receives a TVS NFT through `claimNFT`
2. User attempts to split their TVS NFT by calling `splitTVS(projectId, percentages, nftId)` with valid parameters
3. The `splitTVS` function calls `calculateFeeAndNewAmountForOneTVS` on line 1069 to calculate fees for the split TVS flows
4. `calculateFeeAndNewAmountForOneTVS` attempts to write to `newAmounts[0]` without first allocating the array
5. Solidity panics with error code `0x32` (array out-of-bounds access)
6. The entire transaction reverts, preventing the user from splitting their TVS NFT

### Impact

The NFT holders cannot merge or split their TVS NFTs, resulting in a complete denial of service for both `mergeTVS` and `splitTVS` functionality.

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

contract CalculateFeeBugTest is Test {
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
        // First pass: count how many leaves we need
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leafCount++;
            }
        }

        // CompleteMerkle requires at least 2 leaves, so we need to handle single leaf case
        require(leafCount >= 2, "Need at least 2 bids for Merkle tree");

        // Create properly sized array
        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 currentIndex = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[currentIndex] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);
                bidderPoolIds[bids[i].bidder] = poolId;
                currentIndex++;
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

    /// @notice Test to demonstrate the bug in calculateFeeAndNewAmountForOneTVS
    /// @dev This test proves that the function reverts due to:
    ///      1. newAmounts array is never allocated
    ///      2. Loop variable i is never incremented (infinite loop)
    ///      3. Wrong fee calculation (subtracting cumulative fee from each amount)
    function test_CalculateFeeAndNewAmountForOneTVS_RevertsDueToBugs() public {
        // Set a merge fee rate (100 basis points = 1%)
        vm.prank(projectCreator);
        vesting.setMergeFeeRate(100);

        // Create test amounts array with multiple flows
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 ether;
        amounts[1] = 2000 ether;
        amounts[2] = 3000 ether;
        uint256 length = 3;
        uint256 feeRate = 100; // 1%

        // Attempt to call the function directly
        // This will revert because:
        // 1. newAmounts is never allocated, so writing to newAmounts[i] will cause a panic
        // 2. Even if allocation was fixed, the loop never increments i, causing infinite loop
        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length);
    }

    /// @notice Test to demonstrate the bug through mergeTVS operation
    /// @dev This test proves that mergeTVS reverts when it calls calculateFeeAndNewAmountForOneTVS
    function test_MergeTVS_RevertsDueToCalculateFeeBug() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.setMergeFeeRate(100); // 1% fee

        // 1. Launch project
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create a pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // 3. Place bids (need at least 2 for Merkle tree)
        address bidder = bidders[0];
        address bidder2 = bidders[1];
        
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // 4. Prepare bid allocation
        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        // 5. Generate merkle root
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        // 6. Finalize project
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 7. Claim NFTs for both bidders
        vm.prank(bidder);
        uint256 mergedNftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder]);

        vm.prank(bidder2);
        uint256 nftIdToMerge = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder2]);

        // Verify NFT ownership
        assertEq(nft.ownerOf(mergedNftId), bidder);
        assertEq(nft.ownerOf(nftIdToMerge), bidder2);

        // 8. Attempt to merge TVS - this will revert because calculateFeeAndNewAmountForOneTVS
        //    is called on line 1013 of AlignerzVesting.sol and will fail due to:
        //    - Unallocated newAmounts array
        //    - Infinite loop (i never incremented)
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftIdToMerge;

        vm.prank(bidder);
        // This call will revert because calculateFeeAndNewAmountForOneTVS has critical bugs
        vm.expectRevert();
        vesting.mergeTVS(PROJECT_ID, mergedNftId, projectIds, nftIds);
    }

    /// @notice Test to demonstrate the bug through splitTVS operation
    /// @dev This test proves that splitTVS reverts when it calls calculateFeeAndNewAmountForOneTVS
    function test_SplitTVS_RevertsDueToCalculateFeeBug() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.setSplitFeeRate(100); // 1% fee

        // 1. Launch project
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create a pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        // 3. Place bids (need at least 2 for Merkle tree)
        address bidder = bidders[0];
        address bidder2 = bidders[1];
        
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // 4. Prepare bid allocation
        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        // 5. Generate merkle root
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        // 6. Finalize project
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 7. Claim NFT
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder]);

        // Verify NFT ownership
        assertEq(nft.ownerOf(nftId), bidder);

        // 8. Attempt to split TVS - this will revert because calculateFeeAndNewAmountForOneTVS
        //    is called on line 1069 of AlignerzVesting.sol and will fail due to:
        //    - Unallocated newAmounts array
        //    - Infinite loop (i never incremented)
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        vm.prank(bidder);
        // This call will revert because calculateFeeAndNewAmountForOneTVS has critical bugs
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
}

```

### Mitigation

_No response_
  