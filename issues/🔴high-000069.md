# [000069] _computeSplitArrays() reverts due to uninitialized memory arrays
  
  ## Summary

The [`_computeSplitArrays()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) returns an `Allocation` struct with uninitialized dynamic arrays. This internal function is called by [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088). When the function attempts to assign values to these arrays, it causes an array out-of-bounds error (panic 0x32), making the `splitTVS()` functionality completely unusable.

## Root Cause

The function declares `Allocation memory alloc` as a return value but never initializes its dynamic arrays before attempting to access them:

```solidity
function _computeSplitArrays( Allocation storage allocation, uint256 percentage, uint256 nbOfFlows) internal view returns ( Allocation memory alloc) {
    // ....
    // bug: arrays are never initialized
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;           // reverts here
    // ...
    }
}
```

### The Problem

- `alloc` is a memory struct with uninitialized dynamic arrays
- all arrays (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) have length 0
- attempting to access `alloc.amounts[j]` when the array has length 0 causes a panic error 0x32 (array out-of-bounds)
- this causes the entire `splitTVS()` function to always revert

## Impact

- the `splitTVS()` function is completely broken and will always revert
- users cannot split their tvs (token vesting schedule) tokens
- this is a core functionality that becomes completely unusable
- any user attempting to split their vesting position will lose gas and the transaction will fail

## Proof of Concept

### Test Case

You can add this test to `protocol/test/test.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract test_dos is Test {

    uint256 constant BASIS_POINT = 10_000;
    
    struct Allocation {
        uint256[] amounts;
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
        bool isClaimed;
        IERC20 token;
        uint256 assignedPoolId;
    }
    
    // Storage allocation for testing
    Allocation public testAllocation;
    
    /// @notice copy from AlignerzVesting.sol _computeSplitArrays function (lines 1125-1157)
    /// This is the buggy version with uninitialized arrays
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
    
    /// @notice Public wrapper to call the internal function
    function callComputeSplitArrays(uint256 percentage, uint256 nbOfFlows) public view returns (Allocation memory) {
        return _computeSplitArrays(testAllocation, percentage, nbOfFlows);
    }
    
    /// @notice Test that demonstrates the bug in _computeSplitArrays
    function test_ComputeSplitArrays_Reverts() public {
        // Setup storage allocation with 2 flows
        testAllocation.amounts.push(1000 ether);
        testAllocation.amounts.push(2000 ether);
        
        testAllocation.vestingPeriods.push(90 days);
        testAllocation.vestingPeriods.push(180 days);
        
        testAllocation.vestingStartTimes.push(block.timestamp);
        testAllocation.vestingStartTimes.push(block.timestamp);
        
        testAllocation.claimedSeconds.push(0);
        testAllocation.claimedSeconds.push(0);
        
        testAllocation.claimedFlows.push(false);
        testAllocation.claimedFlows.push(false);
        
        testAllocation.assignedPoolId = 0;
        
        // This will revert with panic 0x32 (array out-of-bounds)
        // because alloc.amounts, alloc.vestingPeriods, etc. are never initialized in _computeSplitArrays
        vm.expectRevert(stdError.indexOOBError);
        this.callComputeSplitArrays(5000, 2);
    }
}
```

### Running the Test

```bash
forge test --match-contract test_dos --match-test test_ComputeSplitArrays_Reverts -vvvv
```

### Trace Output

```
├─ [13299] test_dos::callComputeSplitArrays(5000, 2) [staticcall]
│   └─ ← [Revert] panic: array out-of-bounds access (0x32)
```

## Recommended Fix

Initialize all dynamic arrays with the proper length before accessing them:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (
        Allocation memory alloc
    )
{
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;

    // fix: initialize arrays with proper length
    alloc.amounts = new uint256[](nbOfFlows);
    alloc.vestingPeriods = new uint256[](nbOfFlows);
    alloc.vestingStartTimes = new uint256[](nbOfFlows);
    alloc.claimedSeconds = new uint256[](nbOfFlows);
    alloc.claimedFlows = new bool[](nbOfFlows);

    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked {
            ++j;
        }
    }
}
```

  