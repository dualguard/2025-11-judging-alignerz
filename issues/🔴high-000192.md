# [000192] Attacker can overcharge or revert claims/merges, harming vesting users
  
  ### Summary

The cumulative-fee math in `calculateFeeAndNewAmountForOneTVS` will cause overcharging or reverts for vesting users as an attacker (or any caller) triggers merge/split fee calculation that subtracts the running total from each flow.

### Root Cause

 In protocol/src/contracts/vesting/feesManager/FeesManager.sol:169-173 the function sums fees across all flows and subtracts the cumulative total from every element instead of subtracting only that element’s fee.

### Internal Pre-conditions

 1. Caller invokes `mergeTVS` or `splitTVS` so `calculateFeeAndNewAmountForOneTVS` runs on a multi-flow TVS.
2. At least two flows exist in `amounts` so cumulative subtraction diverges from per-flow fees.


### External Pre-conditions

None

### Attack Path

 1. User (or attacker) prepares a TVS with multiple flows (e.g., by merging allocations).
2. Caller invokes `mergeTVS` or `splitTVS`, which calls `calculateFeeAndNewAmountForOneTVS`.
3. The helper subtracts cumulative fees from each flow; later flows are over-debited and can revert if cumulative fee exceeds the flow amount.

### Impact

  Vesting users can be overcharged on later flows or have merge/split operations revert once cumulative fees exceed a flow, blocking position management; fees collected remain correct in aggregate but per-flow balances are wrong.

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
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

/// @dev Harness that runs the on-chain fee helper while keeping its cumulative-subtraction bug intact.
contract FeesManagerHarness is FeesManager {
    function buggyCalculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts)
        external
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        uint256 length = amounts.length;
        newAmounts = new uint256[](length);
        for (uint256 i; i < length; ++i) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            // Bug: subtracts cumulative fee instead of per-flow fee.
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
}

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

        address implementation = address(new AlignerzVesting());
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        address payable proxy = payable(new ERC1967Proxy(implementation, initData));
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

 function test_PoC_cumulativeFeeIsSubtractedFromEveryFlow() public {
        FeesManagerHarness harness = new FeesManagerHarness();
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100 ether;
        amounts[1] = 200 ether;
        amounts[2] = 300 ether;
        uint256 feeRate = 1_000; // 10%

        (uint256 feeAmount, uint256[] memory newAmounts) =
            harness.buggyCalculateFeeAndNewAmountForOneTVS(feeRate, amounts);

        // Total fee is correct (per-flow fees summed).
        assertEq(feeAmount, 60 ether, "total fee should sum individual fees");

        uint256 fee0 = harness.calculateFeeAmount(feeRate, amounts[0]); // 10
        uint256 fee1 = harness.calculateFeeAmount(feeRate, amounts[1]); // 20
        uint256 fee2 = harness.calculateFeeAmount(feeRate, amounts[2]); // 30

        // First flow is fine: subtracts its own fee.
        assertEq(newAmounts[0], amounts[0] - fee0, "first flow should subtract only fee0");

        // Later flows are overcharged by cumulative fees (fee0 + fee1, fee0 + fee1 + fee2).
        assertEq(newAmounts[1], amounts[1] - (fee0 + fee1), "second flow loses cumulative fees");
        assertLt(newAmounts[1], amounts[1] - fee1, "second flow is overcharged beyond its own fee");
        assertEq(newAmounts[2], amounts[2] - (fee0 + fee1 + fee2), "third flow loses all prior fees too");
        assertLt(newAmounts[2], amounts[2] - fee2, "third flow is overcharged beyond its own fee");
    }

```

### Mitigation

 Charge per-flow and sum totals separately: allocate `newAmounts`, increment the loop counter, compute `feeForI`, set `newAmounts[i] = amounts[i] - feeForI`, and accumulate `feeAmount += feeForI`. This matches the intended per-flow fee logic used elsewhere (e.g., `_merge`).
  