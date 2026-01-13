# [000891] refunds freeze for bidders in nonzero pools
  
  ### Summary

Hardcoding poolId=0 in refund leaf construction will cause refunds in pools with poolId>0 to be unclaimable for bidders as the owner finalizes projects with per-pool refund trees that cannot be proven on-chain.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:835` the refund Merkle leaf is built with a hardcoded `poolId = 0` (`bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));`).

### Internal Pre-conditions

1. Owner launches a bidding project with more than one pool (poolId > 0 exists).
2. Owner finalizes bids with a refund Merkle root that encodes the actual poolId per refunded bidder.
3. A bidder in a nonzero pool attempts to claim their refund via `claimRefund`.

### External Pre-conditions

none

### Attack Path

1. Owner creates multiple pools and later finalizes the project with a refund Merkle tree that includes `poolId` for each refunded bidder.
2. A bidder assigned a refund in `poolId = 1` calls `claimRefund(projectId, amount, proof)` with a proof built off-chain for `(user, amount, projectId, 1)`.
3. On-chain, `claimRefund` force-sets `poolId = 0` before hashing, so the computed leaf differs from the off-chain leaf.
4. `MerkleProof.verify` fails and the refund is permanently unclaimable for that bidder.

### Impact

Bidders in pools with `poolId > 0` cannot claim their stablecoin refunds; funds remain stuck until claimDeadline and may be swept to treasury afterward, causing a total loss of their refundable amounts.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

/// @notice Regression tests for the refund Merkle leaf using a hardcoded poolId (0)
contract AlignerzRefundMerklePoolIdBugTest is Test {
    MockUSD internal token;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal constant TREASURY = address(0xA11CE);
    address internal constant BIDDER_POOL0 = address(0xB0);
    address internal constant BIDDER_POOL1 = address(0xB1);

    uint256 internal constant BID_AMOUNT = 1_000_000; // 1 token with 6 decimals
    uint256 internal constant POOL_ALLOCATION = 10_000_000;

    function setUp() public {
        token = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(TREASURY);
        vesting.setVestingPeriodDivisor(1);

        // Fund bidders with stablecoin for their bids.
        stable.transfer(BIDDER_POOL0, BID_AMOUNT);
        stable.transfer(BIDDER_POOL1, BID_AMOUNT);

        // Approve enough TVS to seed pools.
        token.approve(address(vesting), type(uint256).max);
    }

    function _launchProjectWithTwoPoolsAndBids() internal {
        uint256 start = block.timestamp;
        uint256 end = block.timestamp + 7 days;

        vesting.launchBiddingProject(address(token), address(stable), start, end, bytes32("endHash"), false);

        // Two pools so that poolId 1 is valid.
        vesting.createPool(0, POOL_ALLOCATION, 1, false); // pool 0
        vesting.createPool(0, POOL_ALLOCATION, 1, false); // pool 1

        vm.startPrank(BIDDER_POOL0);
        stable.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(0, BID_AMOUNT, 1);
        vm.stopPrank();

        vm.startPrank(BIDDER_POOL1);
        stable.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(0, BID_AMOUNT, 1);
        vm.stopPrank();
    }

    /// @dev Control: poolId 0 refunds succeed because the hardcoded poolId matches.
    function test_Refund_ForPoolZero_Succeeds() public {
        _launchProjectWithTwoPoolsAndBids();

        bytes32 refundRoot = keccak256(abi.encodePacked(BIDDER_POOL0, BID_AMOUNT, uint256(0), uint256(0)));
        bytes32[] memory poolRoots = new bytes32[](2); // not relevant for refunds

        vesting.finalizeBids(0, refundRoot, poolRoots, 1 days);

        vm.prank(BIDDER_POOL0);
        vesting.claimRefund(0, BID_AMOUNT, new bytes32[](0));

        assertEq(stable.balanceOf(BIDDER_POOL0), BID_AMOUNT, "pool0 refund should succeed");
    }

    /// @dev Regression: poolId > 0 refunds cannot be proven because poolId is hardcoded to 0 in claimRefund.
    function test_Refund_ForPoolOne_RevertsInvalidMerkleProof() public {
        _launchProjectWithTwoPoolsAndBids();

        // Refund leaf for poolId 1 (valid off-chain), but on-chain claimRefund forces poolId = 0.
        bytes32 refundRoot = keccak256(abi.encodePacked(BIDDER_POOL1, BID_AMOUNT, uint256(0), uint256(1)));
        bytes32[] memory poolRoots = new bytes32[](2);

        vesting.finalizeBids(0, refundRoot, poolRoots, 1 days);

        vm.prank(BIDDER_POOL1);
        vm.expectRevert(AlignerzVesting.Invalid_Merkle_Proof.selector);
        vesting.claimRefund(0, BID_AMOUNT, new bytes32[](0));
    }
}
```

### Mitigation

Accept `poolId` as a parameter to `claimRefund` (validated against project.poolCount) and include it in the leaf hash, aligning on-chain verification with off-chain tree generation. If refunds are meant to be pool-agnostic, remove `poolId` from both the on-chain leaf and the off-chain tree schema.
  