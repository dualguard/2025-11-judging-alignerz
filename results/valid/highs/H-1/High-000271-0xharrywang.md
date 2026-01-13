# [000271] Incorrect split calculation in `splitTVS()` causes loss of user funds
  
  ### Summary

The split calculation in `splitTVS()` uses a changing base allocation instead of a fixed original allocation, leading users who call `splitTVS()` to end up with a total allocation that is significantly smaller than the original amount, resulting in permanent loss of funds.

### Root Cause

`splitTVS()` is supposed to take one TVS NFT and split its allocation into several new NFTs according to the given percentages. For each split, it calls `_computeSplitArrays()` to calculate that NFT’s share, and then writes the result back to storage with `_assignAllocation()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107
```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
		...
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
@>          Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
@>          _assignAllocation(newAlloc, alloc);
		...
    }
```

The problem is that all splits share the same allocation storage as their base, and `_computeSplitArrays()` reads from this storage each time:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141
```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
@>      uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
@>          alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

On the first iteration, `allocation.amounts` still holds the full original amounts, so the first split is correct. But `_assignAllocation()` then writes the split result back into storage. On the next iteration, `_computeSplitArrays()` no longer reads the original amounts, it reads the already reduced values from the previous split. This happens again for each further split.

Because each new split is calculated from an amount that was already shrunk by previous splits, every later `alloc.amounts[j]` is too small. In total, the sum of all split TVS allocations is significantly less than the original allocation (ignoring any fees), and the missing part of the user’s funds is lost.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Alice holds a TVS NFT with an allocation of `1000e18` tokens across its flows.
2. Alice calls `splitTVS()` to split this NFT into two NFTs with split percentages `50%` and `50%`. With no fees, the expected result is two NFTs with `500e18` and `500e18` in total.
3. Due to the logic flaw where each iteration uses an already-updated `allocation.amounts` as the base, the first NFT receives `500e18`, but the second NFT is computed from the reduced base and ends up with only `250e18`. 

The final total is `500e18 + 250e18 = 750e18`, so Alice loses `250e18` from her original allocation.

### Impact

Users who split their TVS NFTs through `splitTVS()` will receive less allocation amount. After claiming, they will receive fewer tokens than they should have, resulting in a direct loss of funds.

### PoC

Because `splitTVS()` has several other issues, we need to fix these bugs first to avoid immediate reverts and make this PoC executable.

1.	Replace `calculateFeeAndNewAmountForOneTVS()` in `FeesManager.sol` with the following code, which initializes the dynamic array `newAmounts` and correctly updates the loop index:
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; }
        }
    }
```

2. Replace `_computeSplitArrays()` in `AlignerzVesting.sol` with the following code, which initializes the dynamic arrays in the Allocation struct:
```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        alloc.amounts           = new uint256[](nbOfFlows);
        alloc.vestingPeriods    = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds    = new uint256[](nbOfFlows);
        alloc.claimedFlows      = new bool[](nbOfFlows);

        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

3. Add `getAllocationAmountsHelper()` in `AlignerzVesting.sol` as a helper function to expose `allocations[nftId].amounts` for assertion:
```solidity
    function getAllocationAmountsHelper(bool isBiddingProject, uint256 projectId, uint256 nftId) public returns(uint256[] memory amounts) {
        amounts = isBiddingProject 
            ? biddingProjects[projectId].allocations[nftId].amounts
            : rewardProjects[projectId].allocations[nftId].amounts;
    }
```

These changes ensure that `splitTVS()` has the other bugs fixed so the PoC can run successfully.

Next, to demonstrate the scenario described in the **Attack Path** section, create a `PoC.t.sol` file in the `test` folder with the following code:

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

contract PoC is Test {
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

    // Runs steps 1–9 of the full vesting flow and returns the claimed NFT IDs
    function _vestingFlow() public returns(uint256[] memory nftIds) {
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
        nftIds = new uint256[](15);
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
    }

    // Test entry point
    function test_SplitTVSLosesAllocation() public {
        // Run vesting flow and pick one bidder's NFT
        uint256[] memory nftIds = _vestingFlow();

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        uint256 bidderIndex = 1;
        address splitBidder = bidders[bidderIndex];
        uint256 splitNftId = nftIds[bidderIndex];

        uint256[] memory newNftIds;

        // Verify original allocation amount for this NFT
        uint256 splitNftIdAmountBefore = vesting.getAllocationAmountsHelper(true, PROJECT_ID, splitNftId)[0];
        assertEq(splitNftIdAmountBefore, 1000e18, "initial amount should be 1000e18");

        // Split the TVS into two NFTs with 50/50 percentages
        vm.prank(splitBidder);
        (, newNftIds) = vesting.splitTVS(PROJECT_ID, percentages, splitNftId);

        // Read updated amounts for original and new NFT
        uint256 splitNftIdAmountAfter = vesting.getAllocationAmountsHelper(true, PROJECT_ID, splitNftId)[0];
        uint256 newNftIdAmountAfter = vesting.getAllocationAmountsHelper(true, PROJECT_ID, newNftIds[0])[0];
        
        assertEq(
            splitNftIdAmountAfter + newNftIdAmountAfter, 
            750e18, 
            "total allocation after split is incorrectly reduced"
        );
    }
}
```

Run with
```shell
forge test --mt test_SplitTVSLosesAllocation --force -vvv
```

Output:
```shell
[PASS] test_SplitTVSLosesAllocation() (gas: 19786413)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.67s (5.28ms CPU time)

Ran 1 test suite in 2.68s (2.67s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Mitigation

Redesign the logic in `splitTVS()` and `_computeSplitArrays()` to ensure the split calculation always uses a fixed original base amount that does not change between iterations.
  