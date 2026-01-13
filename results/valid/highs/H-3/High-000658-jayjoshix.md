# [000658] `splitTVS` always reverts due to `_computeSplitArrays` writing into zero-length memory arrays.
  
  ### Summary

Uninitialized dynamic arrays in `_computeSplitArrays` will cause a revert for TVS holders as any caller of `splitTVS` will trigger an array out-of-bounds panic when splitting a TVS, making it impossible for users to split their vesting positions even with valid inputs.

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113C4-L1141C6

In `AlignerzVesting.splitTVS` and `_computeSplitArrays` the split logic uses an `Allocation memory alloc` with dynamic arrays that are never sized before indexed writes, so `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, `alloc.vestingStartTimes[j]`, `alloc.claimedSeconds[j]`, and `alloc.claimedFlows[j]` write into zero-length arrays in a `for` loop, causing a panic `0x32` (array out-of-bounds) on the first iteration.

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (Allocation memory alloc)
{
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;  // out-of-bounds: alloc.amounts has length 0
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked { ++j; }
    }
}
```

Because `Allocation` is a struct with dynamic arrays and `alloc` is a fresh in-memory instance, all of its arrays have length `0` by default, so indexed writes are invalid without prior allocation (e.g., via `new uint256[](nbOfFlows)`).


### Internal Pre-conditions

1. A bidding project must be launched, bids finalized, and at least one TVS NFT claimed so that `allocationOf[nftId]` (and the underlying project allocation) has non-empty `amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, and `claimedFlows` arrays, and `NFTBelongsToBiddingProject[nftId]` is `true`.

2. The caller must own the NFT to be split: `msg.sender == nftContract.extOwnerOf(splitNftId)` must hold, otherwise `splitTVS` would revert earlier with `Caller_Should_Own_The_NFT`.

3. The `percentages` array passed to `splitTVS(projectId, percentages, splitNftId)` must have length greater than `0` (e.g., `2`) so that `nbOfTVS = percentages.length` and `nbOfFlows = allocation.amounts.length` are both non-zero, and the loops in `splitTVS` and `_computeSplitArrays` actually execute.

4. The base allocation for `splitNftId` must have at least one flow, i.e., `allocation.amounts.length > 0`, making `nbOfFlows > 0` and forcing the inner `_computeSplitArrays` loop to run at least once.


### External Pre-conditions

1. No specific external protocol state is required; the bug is purely due to the internal memory allocation logic in `_computeSplitArrays`.

2. Normal gas limits and EVM semantics apply; the panic `0x32` will occur regardless of price or external contract behavior, as it is a Solidity runtime check on array bounds.


### Attack Path

1. A user participates in a bidding project, is accepted, and claims a TVS NFT so that `splitNftId` (e.g., `1`) has a populated `Allocation` with `amounts.length == nbOfFlows > 0`.

2. As the NFT owner, the user calls `splitTVS(projectId, percentages, splitNftId)` with a `percentages` array such as `[5000, 5000]` (two-way split, total 100% in basis points).

3. Inside `splitTVS`, the contract verifies ownership via `nftContract.extOwnerOf(splitNftId)` and retrieves `(Allocation storage allocation, IERC20 token)` for this NFT, with `nbOfFlows = allocation.amounts.length`.

4. The function computes fees and updates `allocation.amounts = newAmounts`, then determines `nbOfTVS = percentages.length` and allocates auxiliary arrays (`newNftIds`, `allAlloc`).

5. In the loop over `i` in `splitTVS`, for each target TVS (including the original `splitNftId` and any newly minted NFTs), it calls `_computeSplitArrays(allocation, percentage, nbOfFlows)` to derive the per-split `Allocation` memory struct.

6. Within `_computeSplitArrays`, a fresh `Allocation memory alloc` is created with *all* dynamic arrays implicitly length `0`, and the code immediately enters a `for (uint256 j; j < nbOfFlows;)` loop.

7. On the first iteration `j = 0`, the function attempts `alloc.amounts[0] = (baseAmounts[0] * percentage) / BASIS_POINT;`, but `alloc.amounts` is a zero-length array, so this indexed write triggers a Solidity runtime panic `0x32` (array index out of bounds).

8. The entire `splitTVS` transaction reverts with this panic, regardless of the validity of `percentages` or the caller’s ownership, and no state changes (including potential fee deductions or NFT mints) are successfully applied.


### Impact

The TVS holder cannot execute `splitTVS` under any valid conditions, as the operation always reverts due to the uninitialized dynamic arrays in `_computeSplitArrays`, effectively making the advertised “split TVS” feature unusable and preventing users from rebalancing or subdividing their vesting positions.

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



function test_DividendDistributor_offByOne_skipsRealNFT() public {
    // Setup: one real TVS NFT with tokenId = 1 via the normal flow
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    vesting.launchBiddingProject(
        address(token),
        address(usdt),
        block.timestamp,
        block.timestamp + 1_000_000,
        bytes32(0),
        true
    );
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

    // Whitelist the first two bidders
    address[] memory someBidders = new address[](2);
    someBidders[0] = bidders[0];
    someBidders[1] = bidders[1];
    vesting.addUsersToWhitelist(someBidders, PROJECT_ID);
    vm.stopPrank();

    // Two bidders place bids into the same pool
    for (uint256 i = 0; i < 2; i++) {
        address bidder = bidders[i];
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();
    }

    // Off‑chain allocation: both bids accepted into pool 0
    BidInfo[] memory allBids = new BidInfo[](2);
    for (uint256 i = 0; i < 2; i++) {
        allBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
    }

    // Merkle roots for allocations and refunds
    bytes32[] memory poolRoots = new bytes32[](1);
    poolRoots[0] = generateMerkleProofs(allBids, 0);
    refundRoot = generateRefundProofs(allBids);

    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

    // bidder0 claims their NFT TVS – this is tokenId = 1
    address bidder0 = bidders[0];
    vm.prank(bidder0);
    uint256 nftId = vesting.claimNFT(
        PROJECT_ID,
        bidderPoolIds[bidder0],
        BIDDER_USD,
        bidderProofs[bidder0]
    );

    assertEq(nftId, 1);
    assertEq(nft.ownerOf(nftId), bidder0);

    // Deploy the harness distributor
    distributor = new A26ZDividendDistributorHarness(
        address(vesting),
        address(nft),
        address(usdt),
        block.timestamp,
        90 days,
        address(0) // mismatched token; we only care about accounting here
    );

    // Seed accounting to pretend there is exactly one NFT with all the unclaimed value
    uint256 totalUnclaimed = 1e18;
    uint256 stableToDistribute = 1_000 ether;

    A26ZDividendDistributorHarness(address(distributor)).seedAccounting(
        nftId,               // nftId = 1
        totalUnclaimed,      // unclaimedAmountsIn[1]
        totalUnclaimed,      // totalUnclaimedAmounts
        stableToDistribute   // stablecoinAmountToDistribute
    );

    // In a correct implementation, the only TVS owner (bidder0) would get
    // all of `stableToDistribute` assigned as dividends.
    // But _setDividends() loops i = 0..len-1 with len = nft.getTotalMinted() == 1,
    // so it only checks tokenId = 0 and NEVER processes tokenId = 1.
    uint256 before = usdt.balanceOf(bidder0);

    vm.prank(address(this)); // test contract is the distributor owner
    distributor.setDividends();

    vm.prank(bidder0);
    distributor.claimDividends();

    uint256 afterBalance = usdt.balanceOf(bidder0);

    // Off‑by‑one bug: holder of tokenId 1 receives nothing,
    // even though all accounting was seeded to that NFT.
    assertEq(
        afterBalance,
        before,
        "Off-by-one: dividends for tokenId 1 were never allocated"
    );
}


function test_SplitTVS_alwaysReverts_dueToUninitializedArrays() public {
    // -----------------------------
    // 1. Setup: bidding project where bidder0 gets one TVS NFT
    // -----------------------------
    vm.startPrank(projectCreator);
    vesting.setVestingPeriodDivisor(1);

    vesting.launchBiddingProject(
        address(token),
        address(usdt),
        block.timestamp,
        block.timestamp + 1_000_000,
        bytes32(0),
        true
    );
    vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

    // Whitelist two bidders so Merkle has >= 2 leaves
    address bidder0 = bidders[0];
    address bidder1 = bidders[1];
    address[] memory someBidders = new address[](2);
    someBidders[0] = bidder0;
    someBidders[1] = bidder1;
    vesting.addUsersToWhitelist(someBidders, PROJECT_ID);
    vm.stopPrank();

    // Two bidders place identical bids into the same pool
    for (uint256 i = 0; i < 2; i++) {
        address bidder = bidders[i];
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();
    }

    // -----------------------------
    // 2. Off-chain allocation: both bids accepted into pool 0
    // -----------------------------
    BidInfo[] memory allBids = new BidInfo[](2);
    for (uint256 i = 0; i < 2; i++) {
        allBids[i] = BidInfo({
            bidder: bidders[i],
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
    }

    bytes32[] memory poolRoots = new bytes32[](1);
    poolRoots[0] = generateMerkleProofs(allBids, 0);
    refundRoot = generateRefundProofs(allBids);

    vm.prank(projectCreator);
    vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

    // -----------------------------
    // 3. bidder0 claims their TVS NFT (tokenId = 1)
    // -----------------------------
    vm.prank(bidder0);
    uint256 nftId = vesting.claimNFT(
        PROJECT_ID,
        bidderPoolIds[bidder0],
        BIDDER_USD,
        bidderProofs[bidder0]
    );

    assertEq(nft.ownerOf(nftId), bidder0);

    // -----------------------------
    // 4. Attempt to split the TVS: even with valid percentages,
    //    splitTVS should revert due to _computeSplitArrays using
    //    zero-length memory arrays.
    // -----------------------------
    uint256[] memory percentages = new uint256[](2);
    percentages[0] = 5000; // 50%
    percentages[1] = 5000; // 50%

    vm.prank(bidder0);
    vm.expectRevert(); // panic 0x32: array out-of-bounds in _computeSplitArrays
    vesting.splitTVS(PROJECT_ID, percentages, nftId);
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

run forge test --ffi --force --mt test_SplitTVS_alwaysReverts_dueToUninitializedArrays -vvvv

### Mitigation

_No response_
  