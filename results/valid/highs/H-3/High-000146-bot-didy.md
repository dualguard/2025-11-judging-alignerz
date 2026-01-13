# [000146] Any user splitting will always revert split operations for NFT holders
  
  ### Summary

Uninitialized split allocation arrays in `_computeSplitArrays` will cause every `splitTVS` call to revert for TVS holders as any holder invokes the split path and immediately hits an array out-of-bounds panic.

### Root Cause

In vesting/AlignerzVesting.sol (`_computeSplitArrays`), the `Allocation memory alloc`’s dynamic arrays are never sized before writes (`alloc.amounts[j] = ...` etc.), so the first write accesses index 0 of zero-length arrays and reverts.

### Internal Pre-conditions

1. A TVS NFT with at least one vesting flow exists (so `allocation.amounts.length > 0`).
2. The caller owns that NFT, allowing them to call `splitTVS`.

### External Pre-conditions

none

### Attack Path

1. NFT holder calls `splitTVS(projectId, percentages, splitNftId)`.
2. The function enters `_computeSplitArrays;` the first write to `alloc.amounts[0]` hits a zero-length array.
3. The transaction reverts with an array out-of-bounds panic; split cannot proceed.

### Impact

 TVS holders cannot split their allocations at all; the split feature is fully DoS’d (no funds stolen, but functionality is broken).

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

// Minimal harness to expose _computeSplitArrays for testing.
contract SplitArraysHarness is AlignerzVesting {
    Allocation internal baseAlloc;

    constructor() {
        // Seed a single flow so base arrays are length 1.
        baseAlloc.amounts.push(1 ether);
        baseAlloc.vestingPeriods.push(30 days);
        baseAlloc.vestingStartTimes.push(block.timestamp);
        baseAlloc.claimedSeconds.push(0);
        baseAlloc.claimedFlows.push(false);
    }

    function callCompute(uint256 percentage) external view returns (Allocation memory) {
        // Calls the vulnerable internal helper directly.
        return _computeSplitArrays(baseAlloc, percentage, baseAlloc.amounts.length);
    }
}

contract SplitArraysBugTest is Test {
    SplitArraysHarness harness;

    function setUp() public {
        harness = new SplitArraysHarness();
    }

    function test_ComputeSplitArraysAlwaysReverts_UninitializedMemoryArrays() public {
        // _computeSplitArrays writes into zero-length memory arrays and panics.
        vm.expectRevert(stdError.indexOOBError);
        harness.callCompute(5_000);
    }
}
```

### Mitigation

- In `_computeSplitArrays`, allocate each dynamic array before writing: `alloc.amounts = new uint256[](nbOfFlows); alloc.vestingPeriods = new uint256[](nbOfFlows); alloc.vestingStartTimes = new uint256[](nbOfFlows); alloc.claimedSeconds = new uint256[](nbOfFlows); alloc.claimedFlows = new bool[](nbOfFlows);`.

- (Optionally) remove `_computeSplitArrays`’s reliance on an uninitialized `Allocation memory` by passing in pre-sized arrays or building the split allocations inline in `splitTVS`.
  