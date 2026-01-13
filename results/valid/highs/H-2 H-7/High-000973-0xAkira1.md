# [000973] The lack of array initialisation in the `calculateFeeAndNewAmountForOneTVS` function causes DoS attack when the `mergeTVS()` and `splitTVS()` functions are called.
  
  ### Summary

Due to incorrect implementation of fee calculation, the `mergeTVS` and `splitTVS` functions will always be unavailable, as there is no array initialization or incrementation in the `calculateFeeAndNewAmountForOneTVS` function. 

### Root Cause

The [FeesManager](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol) contract contains an incorrect implementation of the [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) function. 

Specifically, there is no initialization of the array to which new values are added. Because of this, revert will always be called, which will lead to a constant DoS of the functions that call this function.

Also, even if the array is initialized, it will still constantly return revert OOG because the loop does not contain an increment. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user will want to either merge NFTs or split them.
2. Calls the corresponding functions. 

### Impact

It is not possible to call these functions, which leads to a loss of protocol functionality. Without these functions, users are limited in their actions. 

### PoC

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
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

    uint256 constant NUM_KOLS = 10;
    uint256 constant REWARD_PROJECT_ID = 0;
    uint256 constant VESTING_PERIOD = 365 days; // 1 year vesting
    uint256 constant CLAIM_WINDOW = 30 days; // 30 days to claim
    uint256 constant TOTAL_TVS_ALLOCATION = 1_100_000 ether; // 11M tokens total
    uint256 constant TOKENS_PER_KOL = 100_000 ether; // 100k tokens per KOL

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
        nft = new AlignerzNFT(
            "AlignerzNFT",
            "AZNFT",
            "https://nft.alignerz.bid/"
        );

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
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
    function getLeaf(
        address bidder,
        uint256 amount,
        uint256 projectId,
        uint256 poolId
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(
        BidInfo[] memory bids,
        uint256 poolId
    ) internal returns (bytes32) {
        // First, count how many bids are in this pool
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leafCount++;
            }
        }

        // Create leaves array with correct size
        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 leafIndex = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafIndex] = getLeaf(
                    bids[i].bidder,
                    bids[i].amount,
                    PROJECT_ID,
                    poolId
                );

                bidderPoolIds[bids[i].bidder] = poolId;
                leafIndex++;
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
    function generateRefundProofs(
        BidInfo[] memory bids
    ) internal returns (bytes32) {
        uint256 poolId = 0;

        // First, count how many bids are not accepted
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leafCount++;
            }
        }

        // Create leaves array with correct size
        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 leafIndex = 0;

        // Create leaves for each bid that is not accepted
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leaves[leafIndex] = getLeaf(
                    bids[i].bidder,
                    bids[i].amount,
                    PROJECT_ID,
                    poolId
                );

                bidderPoolIds[bids[i].bidder] = poolId;
                leafIndex++;
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

```

        return root;
    }

    function test_CompleteVestingFlow() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false
        );

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        // vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0)
                ? 90 days
                : (i % 3 == 1)
                    ? 180 days
                    : 365 days;

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
                vestingPeriod: (i % 3 == 0)
                    ? 90 days
                    : (i % 3 == 1)
                        ? 180 days
                        : 365 days,
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
        // Claim NFTs for two different bidders from different pools
        address bidder0 = bidders[0]; // Will own both NFTs
        address bidder1 = bidders[1]; // Will transfer NFT to bidder0

        vm.prank(bidder0);
        uint256 nftId0 = vesting.claimNFT(
            PROJECT_ID,
            0, // poolId 0
            BIDDER_USD,
            bidderProofs[bidder0]
        );

        // Bidder 1 claims NFT from pool 1
        vm.prank(bidder1);
        uint256 nftId1 = vesting.claimNFT(
            PROJECT_ID,
            1, // poolId 1
            BIDDER_USD,
            bidderProofs[bidder1]
        );

        // Transfer NFT from bidder1 to bidder0
        vm.prank(bidder1);
        nft.transferFrom(bidder1, bidder0, nftId1);

        vm.startPrank(bidder0);
        uint256[] memory projectIds = new uint256[](1);
        uint256[] memory nftIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        nftIds[0] = nftId1;

        vm.expectRevert();
        uint256 mergedNftId = vesting.mergeTVS(
            PROJECT_ID,
            nftId0,
            projectIds,
            nftIds
        );
    }

### Mitigation

Initialize the `newAmounts` array in the `calculateFeeAndNewAmountForOneTVS` function.
Also add an increment to avoid an infinite loop.
```solidity 
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory ) {
uint256[] memory newAmounts;
        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); 
            newAmounts[i] = amounts[i] - feeAmount; 
        }
return (feeAmount,newAmounts); 
```
  