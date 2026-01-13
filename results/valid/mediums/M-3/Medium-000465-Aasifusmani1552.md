# [000465] Global `allocationOf` Mapping Not Updated After Token Claims, Leading to Stale & Incorrect NFT State
  
  ### Summary

The `claimTokens` function of `AlignerzVesting` contract updates only the project-specific allocation storage (`biddingProjects[projectId].allocations[nftId] or rewardProjects[...]`), but does not update the global `allocationOf` mapping, which is intended to track the state of each NFT allocation across the system globally.

As a result:

- After a user fully claims their vested tokens and the NFT is burned,
- The project-specific allocation is correctly marked as claimed,
- But `allocationOf[nftId]` still reports `isClaimed = false`, exposing stale, incorrect state through the public interface.

This causes `allocationOf` to become desynchronized from the actual allocation state, producing misleading or incorrect data for any external integrators, UI dashboards, indexers, claim-checkers, or off-chain tools reading global NFT allocation state.

### Root Cause

The root cause of this bug is not updating the global state inside the `claimTokens` function.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

- External applications reading `allocationOf[nftId]` get incorrect data.
- NFTs appear "unclaimed" even after full vesting and burn.
- State inconsistencies accumulate over splits/merges
- Off-chain tools claim calculators behave incorrectly.
- Protocol invariants break:
“Once NFT is fully claimed, allocationOf[nftId].isClaimed must be true.”

This breaks system-wide consistency. And because the `A26ZDividendDistributor` contract uses this data, this magnifies the issue even more.

### PoC

- Please copy/paste the given test in `AlignerzVestingProtocolTest.t.sol` test suite and run with command `forge test --mt test_globalAllocationMappingNotUpdatedPoC -vvvv`

**PoC:-**
```solidity
 function test_globalAllocationMappingNotUpdatedPoC() public {
        address alice = makeAddr("Alice");
        //deal tokens to the users, 1 thousand tokens to each
        deal(address(usdt), alice, 1000e6);
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 3 thousand tokens and 1 usdt price per token
        vesting.createPool(0, 3000 ether, 1e6, false);
        vm.stopPrank();

        /// Place bids for the project
        vm.startPrank(alice);
        usdt.approve(address(vesting), 1000e6);
        vesting.placeBid(0, 1000e6, 90 days);
        vm.stopPrank();

        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](3);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] = BidInfo({bidder: alice, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///users cliam nfts
        vm.prank(alice);
        uint256 nftIdAlice = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[alice]);

        //check if the nft is claimed or not
        assertEq(nft.ownerOf(nftIdAlice), alice);

        //warp timestamp, so that alice can claim tokens
        vm.warp(block.timestamp + 90 days);
        vm.prank(alice);
        vesting.claimTokens(0, nftIdAlice);

        //check if alice claim succeeded
        assertEq(token.balanceOf(alice), 1000e18);

        //still the global allocationOf gives false data
        (bool isClaimed,,) = vesting.allocationOf(nftIdAlice);
        assertEq(isClaimed, false);
    }
```

**Logs:-**
```solidity
 ├─ [3616] ERC1967Proxy::fallback(1) [staticcall]
    │   ├─ [2879] AlignerzVesting::allocationOf(1) [delegatecall]
    │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    ├─ [0] VM::assertEq(false, false) [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.13s (4.39ms CPU time)
```

### Mitigation

Update the global `allocationOf` mapping in the `claimTokens` function.
  