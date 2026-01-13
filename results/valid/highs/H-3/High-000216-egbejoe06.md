# [000216] Users Cannot Split TVS NFTs Due to Unallocated Array Access
  
  ### Summary

A lack of memory allocation for dynamic arrays in `_computeSplitArrays` will cause a denial of service for NFT holders as users attempting to split their TVS NFTs will encounter a panic revert due to array out-of-bounds access.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:1113-1141`, the `_computeSplitArrays` function declares an `Allocation memory alloc` struct but never allocates the dynamic arrays (`alloc.amounts`, `alloc.vestingPeriods`, `alloc.vestingStartTimes`, `alloc.claimedSeconds`, `alloc.claimedFlows`) before attempting to write to them. The function directly indexes these unallocated arrays in a loop, causing a panic revert with error code `0x32` (array out-of-bounds access).
**Code Reference:**
```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    // ... base arrays loaded ...
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
    // ❌ Arrays are never allocated here!
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // Panic: array out-of-bounds
        alloc.vestingPeriods[j] = baseVestings[j]; // Panic: array out-of-bounds
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j]; // Panic: array out-of-bounds
        alloc.claimedSeconds[j] = baseClaimed[j]; // Panic: array out-of-bounds
        alloc.claimedFlows[j] = baseClaimedFlows[j]; // Panic: array out-of-bounds
        unchecked { ++j; }
    }
}
```

### Internal Pre-conditions

1. User needs to own a TVS NFT (obtained through `claimNFT` after placing a bid and project finalization)
2. User needs to call `splitTVS` function with valid parameters (projectId, percentages array, NFT ID)
3. The percentages array must sum to 10000 (BASIS_POINT) - this check happens after the revert, so it's never reached


### External Pre-conditions

_No response_

### Attack Path

1. User places a bid in a bidding project and receives a TVS NFT through `claimNFT`
2. User attempts to split their TVS NFT by calling `splitTVS(projectId, percentages, nftId)` with valid parameters (e.g., splitting into 50/50 with `[5000, 5000]`)
3. The `splitTVS` function calls `_computeSplitArrays` to compute the allocation arrays for each split portion
4. `_computeSplitArrays` attempts to write to `alloc.amounts[0]` without first allocating the array
5. Solidity panics with error code `0x32` (array out-of-bounds access)
6. The entire transaction reverts, preventing the user from splitting their TVS NFT

### Impact

The NFT holders cannot split their TVS NFTs, resulting in a complete denial of service for the `splitTVS` functionality.

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

contract SplitTVSTest is Test {
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

    /// @notice Test to demonstrate the bug in _computeSplitArrays
    /// @dev This test proves that splitTVS always reverts due to unallocated dynamic arrays
    function test_SplitTVS_RevertsDueToUnallocatedArrays() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

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

        // 4. Prepare bid allocation (both accepted, but we'll only claim NFT for bidder)
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

        // 8. Attempt to split TVS - this should revert due to unallocated arrays in _computeSplitArrays
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%
        // Total = 10000 (100% in basis points)

        vm.prank(bidder);
        // This call will revert because _computeSplitArrays tries to write to unallocated dynamic arrays
        // The arrays (alloc.amounts, alloc.vestingPeriods, etc.) are never allocated before being indexed
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
}

```

### Mitigation

_No response_
  