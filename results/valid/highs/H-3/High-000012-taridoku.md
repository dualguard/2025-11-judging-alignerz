# [000012] Uninitialized dynamic arrays in `_computeSplitArrays` bricks `splitTVS`
  
  ### Summary

The `splitTVS` flow relies on `_computeSplitArrays` to construct new Allocation structs for each split TVS. `_computeSplitArrays` returns an Allocation memory with zero-length dynamic arrays and then writes into those arrays without allocating them. This causes an index-out-of-bounds panic for any TVS with at least one flow.

### Root Cause

In `splitTVS`:

```solidity
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
...
_assignAllocation(newAlloc, alloc);
```

[`_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) is then expected to return `Allocation memory alloc`
```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (Allocation memory alloc)
{
...
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token          = allocation.token;

    // BUG: inner arrays of `alloc` are never allocated
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j]           = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j]    = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j]    = baseClaimed[j];
        alloc.claimedFlows[j]      = baseClaimedFlows[j];
        unchecked { ++j; }
    }
}
```
When `Allocation memory alloc` is returned, its dynamic arrays (amounts, vestingPeriods, etc.) are all zero-length by default and have no backing memory. The function never allocates those arrays with `new uint256[](nbOfFlows)` or `new bool[](nbOfFlows)`. Hence, writing `alloc.amounts[j]` for any `j < nbOfFlows` is a write into a zero-length array and causes an index-out-of-bounds panic.


### Internal Pre-conditions

1. The TVS being split has at least one flow (`allocation.amounts.length > 0`).
2. `splitTVS` is called (very normal, user-driven).

### External Pre-conditions

Nil

### Attack Path

Nil

### Impact

Impact level: High
The entire `splitTVS` feature is effectively unusable for all non-empty allocations. This is a full feature-level denial of service: users cannot restructure their TVS positions via splitting.

Likelihood: High
Any attempt to split a TVS with at least one flow will revert. There is no configuration in which splitTVS can operate correctly for realistic allocations.

### PoC

Because of issue #8 , I use a `SplitTVSHarness` for this test. It replicates the function while bypassing the faulty `calculateFeeAndNewAmountForOneTVS`.
Add this test to `./test` as its own suite
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @dev Harness that exposes `_computeSplitArrays` and a storage Allocation
contract SplitTVSHarness is AlignerzVesting {
    // Reuse the same Allocation struct defined in AlignerzVesting
    mapping(uint256 => Allocation) internal testAllocations;

    /// @dev Manually seed an Allocation in storage for testing.
    function setTestAllocation(
        uint256 id,
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes,
        uint256[] memory claimedSeconds,
        bool[] memory claimedFlows,
        IERC20 token,
        uint256 assignedPoolId
    ) external {
        Allocation storage a = testAllocations[id];

        a.amounts = amounts;
        a.vestingPeriods = vestingPeriods;
        a.vestingStartTimes = vestingStartTimes;
        a.claimedSeconds = claimedSeconds;
        a.claimedFlows = claimedFlows;
        a.token = token;
        a.assignedPoolId = assignedPoolId;
    }

    /// @dev Public wrapper that calls the real internal `_computeSplitArrays`.
    /// I don't care about the return value in the test,  just to show the revert.
    function callComputeSplitArrays(
        uint256 id,
        uint256 percentage
    ) external view returns (Allocation memory alloc) {
        Allocation storage a = testAllocations[id];
        uint256 nbOfFlows = a.amounts.length;

        // This calls the buggy internal implementation from AlignerzVesting
        alloc = _computeSplitArrays(a, percentage, nbOfFlows);
    }
}

contract SplitTVSComputeArraysTest is Test {
    SplitTVSHarness internal harness;

    function setUp() public {
        // No need to call initialize; we only use testAllocations + view logic
        harness = new SplitTVSHarness();
    }

    /// @dev S2 PoC:
    /// `_computeSplitArrays` returns an Allocation with zero-length dynamic arrays
    /// and immediately writes to alloc.amounts[j], causing an index-out-of-bounds
    /// panic for any nbOfFlows > 0.
    function test_computeSplitArrays_reverts_dueToUninitializedDynamicArrays() public {
        // --- 1. Build a single-flow Allocation in memory ---
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;

        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 30 days;

        uint256[] memory vestingStartTimes = new uint256[](1);
        vestingStartTimes[0] = block.timestamp;

        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 0;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        // --- 2. Seed this allocation into the harness storage ---
        uint256 allocId = 0;
        harness.setTestAllocation(
            allocId,
            amounts,
            vestingPeriods,
            vestingStartTimes,
            claimedSeconds,
            claimedFlows,
            IERC20(address(0)), // token is unused in _computeSplitArrays
            0                   // assignedPoolId
        );

        // --- 3. Call the wrapper that hits `_computeSplitArrays` ---
        uint256 percentage = 5_000; // 50% – actual value irrelevant for the bug

        // `_computeSplitArrays` will:
        // - create `Allocation memory alloc` with zero-length arrays
        // - immediately do `alloc.amounts[0] = ...;` ->out-of-bounds ->revert
        vm.expectRevert();
        harness.callComputeSplitArrays(allocId, percentage);
    }
}
````

### Mitigation

Allocate inner dynamic arrays in `_computeSplitArrays`:
```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (Allocation memory alloc)
{
    uint256[] memory baseAmounts           = allocation.amounts;
    uint256[] memory baseVestings          = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed           = allocation.claimedSeconds;
    bool[]    memory baseClaimedFlows      = allocation.claimedFlows;

    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token          = allocation.token;

    // Allocate inner arrays before writing
    alloc.amounts           = new uint256[](nbOfFlows);
    alloc.vestingPeriods    = new uint256[](nbOfFlows);
    alloc.vestingStartTimes = new uint256[](nbOfFlows);
    alloc.claimedSeconds    = new uint256[](nbOfFlows);
    alloc.claimedFlows      = new bool[](nbOfFlows);

    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j]           = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j]    = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j]    = baseClaimed[j];
        alloc.claimedFlows[j]      = baseClaimedFlows[j];
        unchecked { ++j; }
    }
}
```
  