# [000394] hasExtraRefund Logic Not Enforced Feature is Non-Functional
  
  ### Summary

.


### Root Cause

The protocol introduces a configuration flag `hasExtraRefund` at the pool level. This flag is intended to grant winning bidders a full refund in addition to receiving their allocated TVS tokens, effectively allowing “free participation” for that specific pool.

However, despite this feature being defined during createPool(), the refund mechanism never checks or uses this variable.

```solidity
.....
biddingProject.vestingPools[poolId] = VestingPool({
            // Merkle root for allocated bids
            merkleRoot: bytes32(0), // Initialize with empty merkle root
            // whether pool refunds the winners as well it allows the user to get a full refund even when his bid got accepted and he got a TVS
            hasExtraRefund: hasExtraRefund
        });
.....
```

Winners cannot claim both NFT + refund
The intended design:
- If a pool has hasExtraRefund, winners:
 - Receive an NFT (via claimNFT)
 - Should also claim a refund (via claimRefund)

But:
The protocol does not link hasExtraRefund to the refund Merkle tree
The protocol does not include a separate “winner-refund” tree
Thus, no winning bidder can ever claim both NFT + refund—breaking the feature completely.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

- A major advertised feature (extra refunds for certain pools) is completely nonfunctional
- Winning users who should receive a refund cannot


### PoC

```solidity
function test_hasExtraRefund_doesNotAutoRefundWinners() public {
        address alice = address(0xA1);
        address bob = address(0xB2);

        // Project and pool IDs
        uint256 projectId = 0;
        uint256 poolId = 0;
        uint256 bidAmount = 100 ether; // bidder commits 100 stablecoins
        uint256 allocationAmount = 1000 ether; // tokens allocated to pool

        // 1) Owner launches a project
        vm.prank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1 days, "0x0", false);

        // 2) Owner creates pool with hasExtraRefund = true
        vm.prank(projectCreator);
        vesting.createPool(projectId, allocationAmount, 1 /*tokenPrice*/, true /*hasExtraRefund*/);
        
        // 3) Alice places a bid during bidding window
        vm.deal(alice, bidAmount);
        usdt.mint(alice, bidAmount);
        vm.startPrank(alice);
        usdt.approve(address(vesting), bidAmount);
        vesting.placeBid(projectId, bidAmount, 1); // vestingPeriod = 0 (instant/no vest)
        vm.stopPrank();

        // 4) Owner finalizes bids to stop bidding
        bytes32[] memory emptyRoots = new bytes32[](1);
        emptyRoots[0] = bytes32(0);
        vm.prank(projectCreator);
        vesting.finalizeBids(projectId, bytes32(0), emptyRoots, 1 hours);

        // === Off-chain backend computes allocations ===
        uint256 allocationForAlice = 50 ether; // the owner decided alice gets 50 tokens
        uint256 refundForBob = 100 ether; // bob (not alice) is in refund tree
        bytes32 allocationLeaf = keccak256(abi.encodePacked(alice, allocationForAlice, projectId, poolId));
        bytes32 refundLeafNonAlice = keccak256(abi.encodePacked(bob, refundForBob, projectId, uint256(0)));

        // 5) Owner publishes merkle roots computed off-chain:
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = allocationLeaf;
        // Refund root intentionally *does not* include alice; it contains bob's leaf only
        bytes32 refundRoot = refundLeafNonAlice;

        vm.prank(projectCreator);
        vesting.updateProjectAllocations(projectId, refundRoot, roots);

        // === Assertions ===
        // 6) Alice can claim her NFT
        bytes32[] memory emptyProof = new bytes32[](0);
        vm.prank(alice);
        uint256 nftId = vesting.claimNFT(projectId, poolId, allocationForAlice, emptyProof);
        assert(nftId != 0);

        // 7) Alice attempts to claim refund, should fail
        vm.prank(alice);
        vm.expectRevert(); // alice cannot calim even the pool is with special condition
        vesting.claimRefund(projectId, allocationForAlice, emptyProof);
    }
```

### Mitigation

_No response_
  