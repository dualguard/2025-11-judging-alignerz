# [000975] 🟠 Medium _computeSplitArrays will always revert due to accessing items in an unititialized array
  
  ### Summary

_computeSplitArrays() returns an Allocation struct which has a number of arrays in it. However, these arrays have 0 length to begin with. When those arrays are indexed into, the function will fail no matter what.

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
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT; // <- reading into uninitialized array
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

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

NA

### Impact

splitTVS() will always fail because of this

### PoC

_No response_

### Mitigation

Append to the arrays rather than index into a nonexistent slot
  