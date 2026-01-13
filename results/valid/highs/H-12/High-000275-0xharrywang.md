# [000275] Wrong check in `getUnclaimedAmounts()` causes incorrect unclaimed totals
  
  ### Summary

`getUnclaimedAmounts()` has a check flaw: the token comparison condition is reversed. TVSs that actually match the target token are skipped and treated as having zero unclaimed amount. This leads to incorrect total unclaimed token calculations and can break downstream dividend logic.

### Root Cause

`getUnclaimedAmounts()` in `A26ZDividendDistributor` is used to query the unclaimed amount of tokens for a given TVS NFT. By design, the dividend distributor should only account for TVSs whose underlying token matches the configured token in the distributor.

However, the token equality check is inverted. When the TVS token matches the distributor token, the function immediately returns `0,` skipping the calculation that should be performed for matching TVSs:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
@>      if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        ...
    }
```

As a result, all TVSs that should be included in the unclaimed token computation are incorrectly skipped, and their unclaimed balances are treated as zero. This causes underestimation (often to zero) of unclaimed totals and corrupts the dividend distribution logic that depends on these values.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.	The protocol launches a vesting bidding project for `tokenA`. Through the normal bidding and distribution flow, eligible bidders successfully call `claimNFT()` and receive TVS NFTs that lock `tokenA`.
2.	The owner performs a dividend operation and calls `A26ZDividendDistributor.setAmounts()` to set the amounts for the dividend distribution. Internally, this relies on `getTotalUnclaimedAmounts()`, which aggregates results from `getUnclaimedAmounts()` for each TVS. Because of the reversed condition, all TVSs that actually lock `tokenA` are treated as having `0` unclaimed amount, so `getTotalUnclaimedAmounts()` returns `0`.
3.	Optionally, if the owner calls `setDividends()`, the internal math may use `totalUnclaimedAmounts` as a denominator. Since `totalUnclaimedAmounts` is incorrectly `0`, this can lead to a division-by-zero revert, causing the dividend operation to fail completely.

### Impact

`getUnclaimedAmounts()` miscalculates the unclaimed token amounts, which leads to incorrect total unclaimed token values (often `0`) used in dividend configuration and potential DoS of the dividend setup flow.

### PoC

Because `getUnclaimedAmounts()` also suffers from an ABI mismatch issue with `allocationOf()`, that bug must be fixed first to avoid immediate reverts and make this PoC executable.

First, add a dedicated getter in `AlignerzVesting.sol` to replace the auto-generated mapping getter for `allocationOf`:
```diff
+   function getAllocationOf(uint256 nftId) public view returns (Allocation memory) {
+       return allocationOf[nftId];
+   }
```

Then add the corresponding interface in `IAlignerzVesting.sol`:
```diff
+   function getAllocationOf(uint256 nftId) external view returns (Allocation memory);
```

Update `getUnclaimedAmounts()` in `A26ZDividendDistributor.sol` to use `getAllocationOf()` instead of the auto-generated getter:
```diff
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-       if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
-       uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
-       uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
-       uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
-       bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
-       uint256 len = vesting.allocationOf(nftId).amounts.length;
        
+       if (address(token) == address(vesting.getAllocationOf(nftId).token)) return 0;
+       uint256[] memory amounts = vesting.getAllocationOf(nftId).amounts;
+       uint256[] memory claimedSeconds = vesting.getAllocationOf(nftId).claimedSeconds;
+       uint256[] memory vestingPeriods = vesting.getAllocationOf(nftId).vestingPeriods;
+       bool[] memory claimedFlows = vesting.getAllocationOf(nftId).claimedFlows;
+       uint256 len = vesting.getAllocationOf(nftId).amounts.length;
        ...
    }
```
This ensures getUnclaimedAmounts() no longer reverts due to ABI mismatch, so we can run this PoC normally.

Now, to demonstrate the scenario described in the **Attack Path** section, create a test file `PoC.t.sol` in the `test` folder with the following code and run it with:

`forge test --mt test_GetTotalUnclaimedAmountsSkipsMatchingTokenTVS --force -vvv`

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

contract PoC is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    
    A26ZDividendDistributor public distributor;

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

        vm.prank(owner);
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp, 
            block.timestamp + 1_000_000,
            address(token)
        );

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

    // Test entry point
    function test_GetTotalUnclaimedAmountsSkipsMatchingTokenTVS() public {
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
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // First test getTotalUnclaimedAmounts() solo, incorrectly returns 0 due to reversed token check
        uint256 totalUnclaimedAmounts = distributor.getTotalUnclaimedAmounts();
        assertEq(totalUnclaimedAmounts, 0, "totalUnclaimedAmounts should incorrectly be 0");

        // 10. Owner calls setAmounts(), which will be based on this incorrect 0 total
        vm.prank(owner);
        distributor.setAmounts();
        // After update via setAmounts(), totalUnclaimedAmounts value
        totalUnclaimedAmounts = distributor.totalUnclaimedAmounts();
        assertEq(totalUnclaimedAmounts, 0, "totalUnclaimedAmounts should still incorrectly be 0");

    }
}
```


### Mitigation

The token comparison in `getUnclaimedAmounts()` should be inverted so that only TVSs whose `Allocation.token` matches the distributor token are processed, and others are immediately treated as having zero unclaimed amount:
```diff
-       if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+       if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```

  