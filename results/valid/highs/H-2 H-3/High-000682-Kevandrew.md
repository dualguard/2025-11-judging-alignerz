# [000682] [H-3] Merge/Split vesting NFTs always revert (fee helper and split math broken)
  
  ### Summary

The merge/split flows call `calculateFeeAndNewAmountForOneTVS`, which never allocates `newAmounts` and never increments its loop index, so it reverts/OOG; additionally, `_computeSplitArrays` writes into zero-length arrays. As shipped, both `mergeTVS` and `splitTVS` are unusable and always revert.

### Root Cause

  - In `FeesManager.sol:169-173`, `calculateFeeAndNewAmountForOneTVS` uses `for (uint256 i; i < length;)` with no `++i` and writes to `newAmounts[i]` without allocating `newAmounts`.
  - In `AlignerzVesting.sol:1129-1136`, `_computeSplitArrays` writes into memory arrays (`alloc.amounts[j]`, etc.) that are never sized to `nbOfFlows`, so even if the fee helper were fixed, `splitTVS` would still revert.

### Internal Pre-conditions

  1. A vesting NFT exists (normal flow).
  2. Caller invokes `mergeTVS` or `splitTVS` as intended.

### External Pre-conditions

None.

### Attack Path

  1. User calls `mergeTVS` or `splitTVS`.
  2. Function enters `calculateFeeAndNewAmountForOneTVS`; loop lacks `++i` and `newAmounts` is zero-length → reverts/OOG immediately.
  3. (Even if helper fixed) `_computeSplitArrays` writes into zero-length arrays and reverts.

### Impact

Merge and split functionality is completely broken; users cannot consolidate or split vesting NFTs. UX and composability are blocked; any call reverts.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";

contract MergeSplitFeeTest is Test {
    AlignerzVesting private vesting;
    AlignerzNFT private nft;
    Aligners26 private token;
    MockUSD private usdt;

    uint256 private projectId;
    address private ownerAddr;
    address private bidder;

    function setUp() public {
        ownerAddr = address(this);
        bidder = makeAddr("bidder");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));
        vesting.transferOwnership(ownerAddr);

        // fund and approve
        token.transfer(ownerAddr, token.balanceOf(address(this)));
        token.approve(address(vesting), type(uint256).max);
        usdt.mint(bidder, 1_000 ether);
        vm.prank(bidder);
        usdt.approve(address(vesting), type(uint256).max);

        // launch project and pool
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1000, "0x0", false);
        projectId = 0;
        vesting.createPool(projectId, 1_000 ether, 1 ether, false);
        vesting.setVestingPeriodDivisor(1);

        // place bid and finalize with simple root to allow mint
        vm.prank(bidder);
        vesting.placeBid(projectId, 100 ether, 1);
        bytes32[] memory roots = new bytes32[](1);
        // trivial root: leaf hash of bidder allocation into pool 0
        bytes32 leaf = keccak256(abi.encodePacked(bidder, uint256(100 ether), projectId, uint256(0)));
        roots[0] = leaf;
        vesting.finalizeBids(projectId, bytes32(0), roots, 1 days);
        // claim NFT
        bytes32[] memory proof = new bytes32[](0); // root == leaf, empty proof succeeds
        vm.prank(bidder);
        vesting.claimNFT(projectId, 0, 100 ether, proof);
    }

    function test_mergeTVS_revertsDueToNonAllocatingFeeHelper() public {
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = projectId;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = 0;
        vm.prank(bidder);
        vm.expectRevert(); // fee helper has infinite loop / zero-length writes
        vesting.mergeTVS(projectId, 0, projectIds, nftIds);
    }

    function test_splitTVS_revertsDueToNonAllocatingFeeHelper() public {
        uint256[] memory percentages = new uint256[](1);
        percentages[0] = 10_000; // 100%
        vm.prank(bidder);
        vm.expectRevert(); // fee helper has infinite loop / zero-length writes
        vesting.splitTVS(projectId, percentages, 0);
    }
}
```

### Mitigation

  1. In `calculateFeeAndNewAmountForOneTVS`, allocate `newAmounts = new uint256[](length)` and increment `i` each loop.
  2. In `_computeSplitArrays`, allocate per-field arrays to `nbOfFlows` before writing.
  3. Add regression tests covering merge/split happy paths.
  