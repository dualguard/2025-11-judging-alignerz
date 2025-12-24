# [000044] Uninitialized Dynamic Arrays in Allocation Struct
  
  ### Summary


The  `_computeSplitArrays()` function doesn't initialize the arrays in the returned Allocation memory - it only sets `alloc.amounts[j]` but the arrays are not initialized with proper length this will cause revert.

### Root Cause

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

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1121



### Attack Path

Solidity memory structs **do not auto-allocate lengths** for dynamic arrays.
The developer must explicitly allocate:

```solidity
alloc.amounts = new uint256[](nbOfFlows);
```

Without this, the struct’s arrays remain empty (`length = 0`), and indexing them is invalid.


### Impact


`splitTVS` will always revert whenever it attempts to populate new split allocations.


### PoC

**run this in Remix IDE**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract PoC_UninitializedAllocation {
    struct Allocation {
        uint256[] amounts;
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
    }

    function testBug() external pure {
        Allocation memory alloc;

        // This will revert because alloc.amounts has length = 0
        alloc.amounts[0] = 123;
    }
}


```

### Mitigation


Inside `_computeSplitArrays`, initialize all arrays before writing to them:

```solidity
alloc.amounts = new uint256[](nbOfFlows);
alloc.vestingPeriods = new uint256[](nbOfFlows);
alloc.vestingStartTimes = new uint256[](nbOfFlows);
alloc.claimedSeconds = new uint256[](nbOfFlows);
alloc.claimedFlows = new bool[](nbOfFlows);
```
  