# [000895] splitTVS is unusable (out-of-bounds panic)
  
  

## Summary
**splitTVS always reverts**. `_computeSplitArrays` builds a memory `Allocation` but never allocates its dynamic arrays. The first indexed write (`alloc.amounts[j]`, etc.) triggers an out-of-bounds panic, so every splitTVS call fails before any logic executes.

## Root Cause
In `protocol/src/contracts/vesting/AlignerzVesting.sol`, `_computeSplitArrays` returns an uninitialized memory `Allocation` and immediately indexes into zero-length arrays:
```solidity
Allocation memory alloc;
...
alloc.amounts[j] = ...; // amounts is zero-length -> index OOB panic
```
Because arrays are never allocated with `new uint256[](nbOfFlows)` / `new bool[](nbOfFlows)`, any call path that reaches this write reverts.

## Internal Pre-conditions
1. Caller owns a TVS NFT and invokes `splitTVS` with any valid percentages array.
2. `_computeSplitArrays` is entered with `nbOfFlows > 0` (always true for a TVS with at least one flow).

## External Pre-conditions
- None.

## Attack / Failure Path
1. User calls `splitTVS` on a legitimate NFT (e.g., after claiming a reward/bid allocation).
2. `splitTVS` invokes `_computeSplitArrays`.
3. `_computeSplitArrays` writes into zero-length memory arrays and panics with `index out of bounds`.
4. The transaction fully reverts; no state changes or events occur.

## Impact
- Split functionality is completely unavailable; users cannot restructure vesting positions into smaller NFTs.
- Frontends/workflows that rely on splits are broken; no `TVSSplit` events are ever emitted.
- Masks other flow-multiplication vectors (e.g., future flow-explosion DoS) behind a hard revert, hiding real DoS risk once fixed.

## PoC
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

/// @notice Demonstrates that splitTVS always reverts because `_computeSplitArrays`
///         writes into zero-length memory arrays (out-of-bounds panic).
contract AlignerzSplitArraysPanicTest is Test {
    MockUSD internal tvs;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal user = address(0xBEEF);
    uint256 internal constant ONE_TOKEN = 1_000_000; // 1 token with 6 decimals

    function setUp() public {
        tvs = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));

        // Let vesting pull funds for allocations
        tvs.approve(address(vesting), type(uint256).max);
        stable.approve(address(vesting), type(uint256).max);

        vesting.setTreasury(address(1));
    }

    function test_Split_RevertsWithIndexOOB() public {
        // Set up a simple reward project and mint one vesting NFT to the user
        uint256 start = block.timestamp;
        vesting.launchRewardProject(address(tvs), address(stable), start, 30 days);

        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = ONE_TOKEN;
        vesting.setTVSAllocation(0, ONE_TOKEN, 30 days, kols, amounts);

        vm.prank(user);
        vesting.claimRewardTVS(0);
        uint256 nftId = 1; // ERC721A starts at 1

        // Attempt a valid-looking split (50/50) should panic on zero-length memory arrays
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5_000;
        percentages[1] = 5_000;

        uint256 supplyBefore = nft.totalSupply();

        vm.expectRevert(stdError.indexOOBError); // out-of-bounds panic from _computeSplitArrays
        vm.prank(user);
        vesting.splitTVS(0, percentages, nftId);

        // No new NFTs are minted because the call reverted
        assertEq(nft.totalSupply(), supplyBefore, "split should not change totalSupply on revert");
    }
}
```
Key steps (see `protocol/test/AlignerzSplitArraysPanic.t.sol`):
- Set up a reward project and mint a vesting NFT to the user.
- Attempt a 50/50 split.
- Expect `stdError.indexOOBError` (out-of-bounds panic) and observe `totalSupply` unchanged (no side effects).

## Mitigation
- Allocate memory arrays before writing:
  ```solidity
  alloc.amounts = new uint256[](nbOfFlows);
  alloc.vestingPeriods = new uint256[](nbOfFlows);
  alloc.vestingStartTimes = new uint256[](nbOfFlows);
  alloc.claimedSeconds = new uint256[](nbOfFlows);
  alloc.claimedFlows = new bool[](nbOfFlows);
  ```
- Alternatively, construct the split directly in storage without an intermediate unallocated memory struct.

  