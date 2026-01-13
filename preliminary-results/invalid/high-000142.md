# [000142] Refund claims fail for pools > 0
  
  ### Summary

claimRefund hashes a hard-coded poolId = 0, so any refund Merkle tree that encodes the actual pool IDs for pools 1..N cannot be proven; bidders in those pools can never claim their refunds.

### Root Cause

claimRefund sets uint256 poolId = 0; when computing the leaf (keccak256(msg.sender, amount, projectId, poolId)), ignoring the real pool.

### Internal Pre-conditions

(1) Refund root includes pool IDs for pools > 0; 
(2) bidder placed a bid and is eligible for refund; 
(3) owner finalized with that refund root.

### External Pre-conditions

_No response_

### Attack Path

Bidder in pool 1 calls claimRefund; leaf computed with poolId 0 mismatches the root; MerkleProof.verify fails; refund is unclaimable.

### Impact

High for affected users — refunds for pools 1..N are impossible, locking their stablecoins.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test} from "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";

/// @dev PoC: claimRefund hard-codes poolId=0 in the leaf, so refunds for other pools cannot be claimed.
contract RefundPoolIdBugTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    MockUSD stable;

    address bidder = address(this);

    function setUp() public {
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri/");
        token = new Aligners26("Aligners", "ALN");
        stable = new MockUSD();

        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Launch bidding project with 2 pools.
        vesting.launchBiddingProject(address(token), address(stable), block.timestamp, block.timestamp + 10 days, bytes32(0), false);
        vesting.createPool(0, 1 ether, 0.01 ether, false); // pool 0
        vesting.createPool(0, 1 ether, 0.02 ether, false); // pool 1

        // Fund bidder and place bid (pool assignment irrelevant for refund proof).
        stable.mint(bidder, 10 ether);
        vm.prank(bidder);
        stable.approve(address(vesting), 10 ether);
        vm.prank(bidder);
        vesting.placeBid(0, 10 ether, vesting.vestingPeriodDivisor()); // vestingPeriod divisible by divisor

        // Finalize with a refund root that encodes poolId = 1 in the leaf.
        uint256 poolId = 1;
        bytes32 refundLeaf = keccak256(abi.encodePacked(bidder, 10 ether, uint256(0), poolId));
        bytes32 refundRoot = refundLeaf; // single-leaf tree, proof is empty
        bytes32[] memory merkleRoots = new bytes32[](2); // pool roots (not used in claimRefund)
        merkleRoots[0] = bytes32(0);
        merkleRoots[1] = bytes32(0);
        vesting.finalizeBids(0, refundRoot, merkleRoots, 30 days);
    }

    function test_RefundForPoolOneCannotBeClaimed() public {
        // claimRefund hard-codes poolId=0 in the leaf, so verification fails for poolId=1 refunds.
        vm.expectRevert(AlignerzVesting.Invalid_Merkle_Proof.selector);
        vesting.claimRefund(0, 10 ether, new bytes32[](0));
    }
}
```

### Mitigation

_No response_
  