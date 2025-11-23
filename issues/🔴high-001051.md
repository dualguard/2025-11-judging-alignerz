# [001051] Refunds are denied to users with non-zero pools
  
  ### Summary

In `AlignerzVesting::claimRefund`, the function builds the merkle leaf with a hardcoded poolId = 0.
Off‑chain merkle proofs that encode a non‑zero poolId will fail verification and causing `claimRefund` to revert.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835-L853

### Root Cause

`claimRefund` hardcodes `uint256 poolId = 0;` when it builds the merkle leaf:
```solidity
    function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
        // ...
        //@audit poolid != 0 will revert refund
 >>     uint256 poolId = 0;
        // Verify merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
        require(!claimedRefund[leaf], Already_Claimed());
        require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
        claimedRefund[leaf] = true;
        // ....
    }
```
Elsewhere like in `claimNFT` poolId is used in the leaf hash, so mixing a hardcoded poolId=0 in refunds is inconsistent and unlikely for multi-pool bidding projects. 

### Internal Pre-conditions

1. Bidding project deadline should be passed.
2. User has to have rejected bids.

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Refunds for bidders allocated or refunded into non‑zero pools will be DOSed to claim.

### PoC

Add the following test in `AlignerzVestingProtocolTest` and run ` forge test --mt test_ClaimRefund_Fails_For_NonZeroPool -vvvv`

```solidity
    function test_ClaimRefund_Fails_For_NonZeroPool() public {
        // Setup a project with two pools and one bidder
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true
        );
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 1_000_000 ether, 0.02 ether, true);
        vm.stopPrank();

        // whitelist bidder[0]
        address bidder = bidders[0];
        address[] memory wl = new address[](1);
        wl[0] = bidder;
        vm.prank(projectCreator);
        vesting.addUsersToWhitelist(wl, PROJECT_ID);

        // bidder places a bid
        vm.prank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vm.prank(bidder);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);

        // Build a refund merkle root using poolId = 1 (non-zero) so the leaf encodes poolId=1
        // CompleteMerkle::getRoot reverts for a single-leaf array in this implementation,
        // so add a dummy zero leaf to make length >= 2 while keeping the target leaf at index 0.
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(bidder, BIDDER_USD, PROJECT_ID, 1);
        leaves[1] = bytes32(0);
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof = m.getProof(leaves, 0);

        // Finalize project with this refund root (and empty pool roots)
        bytes32[] memory poolRoots = new bytes32[](2);
        poolRoots[0] = bytes32(0);
        poolRoots[1] = bytes32(0);
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, root, poolRoots, 60);

        // Now attempt to claim refund using the proof for poolId=1
        // The contract's `claimRefund` uses poolId = 0 when building the leaf, so verification should fail
        vm.prank(bidder);
        vm.expectRevert();
        vesting.claimRefund(PROJECT_ID, BIDDER_USD, proof);
    }
```

### Mitigation

Match `claimNFT` pattern. Modify `claimRefund` to accept `uint256 poolId` and use it in the leaf.
  