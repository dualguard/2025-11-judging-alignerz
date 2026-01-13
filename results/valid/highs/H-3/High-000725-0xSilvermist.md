# [000725] Uninitialized dynamic arrays in `_computeSplitArrays` causes DoS
  
  ### Summary

The `_computeSplitArrays` function in `AlignerzVesting.sol` returns an `Allocation` memory struct with uninitialized dynamic arrays, then attempts to write to these arrays, causing all calls to `splitTVS` to revert. This completely breaks the TVS splitting functionality, preventing users from splitting their vesting NFTs and rendering a core protocol feature unusable.

### Root Cause

In the `_computeSplitArrays` function, the returned `Allocation memory alloc` struct contains dynamic arrays (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) that are never initialized with proper length before being accessed.

In Solidity, when you declare a memory struct with dynamic arrays, those arrays are initialized with zero length. Attempting to write to `alloc.amounts[j]` when `alloc.amounts.length == 0` will cause an out-of-bounds access revert.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User owns a vesting NFT from any project
2. User calls `splitTVS` to split their NFT.
3. Function enters the loop at `splitTVS`
4. First iteration (i = 0): calls `_computeSplitArrays`
5. Inside `_computeSplitArrays`, the loop tries to execute: `alloc.amounts[0] = ...`
6. Since `alloc.amounts` has length 0 (uninitialized), this causes an out-of-bounds revert

### Impact

All calls to `splitTVS` will revert, making it impossible for any user to split their vesting NFTs.

### PoC

_No response_

### Mitigation


Initialize all dynamic arrays in the `_computeSplitArrays` function before attempting to write to them:

```diff
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
    
+    alloc.amounts = new uint256[](nbOfFlows);
+    alloc.vestingPeriods = new uint256[](nbOfFlows);
+    alloc.vestingStartTimes = new uint256[](nbOfFlows);
+    alloc.claimedSeconds = new uint256[](nbOfFlows);
+    alloc.claimedFlows = new bool[](nbOfFlows);
    
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

  