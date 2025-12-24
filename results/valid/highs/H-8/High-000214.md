# [000214] Dividend Distributor Has Infinite Loop in getUnclaimedAmounts
  
  ### Summary

The `getUnclaimedAmounts` function contains a critical infinite loop bug where the loop variable `i` is not incremented when `continue` is executed. When a flow is marked as claimed (`claimedFlows[i] == true`), the function calls `continue` without incrementing `i`, causing the loop to check the same index repeatedly until all gas is consumed and the transaction reverts.

### Root Cause

In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:147-159`, the `getUnclaimedAmounts` function uses a loop with `continue` statements but fails to increment the loop variable `i` before calling `continue`. This causes an infinite loop when `claimedFlows[i]` is `true`.

**The Problem:**
- When `claimedFlows[i]` is `true`, the function calls `continue` without incrementing `i`
- The loop immediately checks `claimedFlows[i]` again with the same `i` value
- This creates an infinite loop that consumes all gas and causes the transaction to revert
- The same issue exists when `claimedSeconds[i] == 0` - `continue` is called without incrementing `i`

### Internal Pre-conditions

1. A TVS NFT must exist (obtained through `claimNFT` after placing a bid and project finalization)
2. The TVS NFT must have at least one flow marked as claimed (`claimedFlows[i] == true`)
   - This happens when a user has claimed tokens from their vesting schedule
3. The dividend distributor must be configured with a different token than the TVS token (to bypass the early return on line 141)
4. Someone calls `getUnclaimedAmounts(nftId)` or `getTotalUnclaimedAmounts()` which internally calls `getUnclaimedAmounts`

### External Pre-conditions

_No response_

### Attack Path

1. User places a bid in a bidding project and receives a TVS NFT through `claimNFT`
2. User claims some tokens from their vesting schedule by calling `claimTokens`, which marks some flows as claimed (`claimedFlows[i] = true`)
3. Owner or user attempts to calculate unclaimed dividend amounts by calling:
   - `getUnclaimedAmounts(nftId)` directly, OR
   - `getTotalUnclaimedAmounts()` which calls `getUnclaimedAmounts` for all NFTs, OR
   - `setAmounts()` which calls `getTotalUnclaimedAmounts()`
4. The `getUnclaimedAmounts` function enters the loop and encounters a flow where `claimedFlows[i] == true`
5. The function calls `continue` without incrementing `i`
6. The loop immediately checks `claimedFlows[i]` again with the same `i` value
7. This creates an infinite loop that consumes all gas
8. The transaction reverts with an out-of-gas error, preventing dividend calculations


### Impact

The dividend distribution system is completely broken for any TVS NFT that has claimed flows. The impact affects:

- **All users** who have claimed tokens from their vesting schedule - they cannot calculate unclaimed dividend amounts
- **Protocol functionality** - the entire dividend distribution system is non-functional for users with claimed flows
- **Owner operations** - the owner cannot call `setAmounts()` or `setUpTheDividends()` if any TVS has claimed flows, breaking the entire dividend setup process
- **User experience** - users cannot claim dividends because the system cannot calculate their unclaimed amounts

The function will always revert with an out-of-gas error whenever it encounters a claimed flow, making dividend distribution completely unusable for the majority of users who have interacted with their vesting schedules.

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
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract DividendDistributorBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public dividendDistributor;

    address public owner;
    address public projectCreator;
    address public bidder;

    // Constants
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        bidder = makeAddr("bidder");
        vm.deal(projectCreator, 100 ether);
        vm.deal(bidder, 50 ether);

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

        // Setup bidder
        usdt.mint(bidder, BIDDER_USD);

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address _bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(_bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leafCount++;
            }
        }

        require(leafCount >= 2, "Need at least 2 bids for Merkle tree");

        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 currentIndex = 0;

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

    /// @notice Test to demonstrate the infinite loop bug in getUnclaimedAmounts
    /// @dev When a flow is claimed, continue is called without incrementing i, causing infinite loop
    function test_GetUnclaimedAmounts_InfiniteLoop() public {
        // Setup project and claim NFT
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        
        address[] memory whitelist = new address[](2);
        whitelist[0] = bidder;
        whitelist[1] = makeAddr("bidder2");
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        address bidder2 = whitelist[1];
        // Mint USDT from owner (test contract) before pranking as bidder2
        usdt.mint(bidder2, BIDDER_USD);
        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

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

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidder]);

        // Deploy dividend distributor with DIFFERENT token to avoid the first bug
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(differentToken) // Different token to bypass token filter bug
        );

        // Fast forward time and claim some tokens to mark a flow as claimed
        vm.warp(block.timestamp + 30 days);
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);

        // Bug: If any claimedFlows[i] is true, the loop will continue without incrementing i
        // This causes an infinite loop that will consume all gas and revert
        // The function tries to write to unclaimedAmountsIn[nftId] but never reaches it due to infinite loop
        vm.expectRevert(); // Will revert due to out of gas from infinite loop
        dividendDistributor.getUnclaimedAmounts(nftId);
    }
}



```

### Mitigation

_No response_
  