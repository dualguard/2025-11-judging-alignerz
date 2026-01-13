# [000219] Splitting a TVS always leads to revert due to uninitialized dynamic arrays in _computeSplitArrays()
  
  ### Summary

The lack of dynamic array initialization in `_computeSplitArrays()` will cause irreversible corruption of TVS allocations, resulting in a complete loss of vesting functionality for users, as splitting a TVS will always revert due to out-of-bounds memory writes.

### Root Cause

When splitting a TVS, the contract calls:
```js
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
```

Inside `_computeSplitArrays()`, the function attempts to write into uninitialized dynamic arrays inside the returned `Allocation memory alloc`:
```js
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
        ...

        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;  //@audit - arrays are not initialized (no memory allocation)
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
But none of these internal arrays are allocated with a length:
```js
alloc.amounts       // length = 0
alloc.vestingPeriods // length = 0
alloc.vestingStartTimes // length = 0
alloc.claimedSeconds // length = 0
alloc.claimedFlows // length = 0
```
Thus, every line alloc.X[j] = ... writes out of bounds, causing the `splitTVS()` function to always revert.
This makes any call to `splitTVS()` impossible, corrupts the returned allocations, and prevents users from creating valid vesting flows.

### Internal Pre-conditions

1. A user must hold a valid TVS NFT with a non-empty allocation (i.e., allocation.amounts.length > 0).
2. User calls `splitTVS(projectId, percentages, splitNftId)`.

### External Pre-conditions

None.

### Attack Path

1. User calls `splitTVS()` to split their vesting NFT.
2. Function reaches `_computeSplitArrays()`.
3. The returned Allocation memory alloc contains uninitialized dynamic arrays.
4. Any write `alloc.X[j] = ...` attempts to write to index j where length = 0.
5. Execution reverts with out-of-bounds memory write.
6. TVS splitting permanently fails for all users.
7. Any flow relying on split allocations (including merges, claims, and downstream vesting logic) becomes unusable.

### Impact

Permanent DoS of TVS splitting functionality which breaks a core protocol feature described in the whitepaper.

### PoC

Paste the code below inside Remix IDE.
The PoC demonstrates how a call to `_computeSplitArrays()` reverts due to unallocated dynamic arrays.

```js
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

contract Split {
    struct Allocation {
        uint256[] amounts; 
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
        bool isClaimed; 
        uint256 assignedPoolId;
    }

    Allocation public allocStorage;

    constructor() {
        // Prepare a real allocation in storage
        allocStorage.amounts = new uint256[](1);
        allocStorage.vestingPeriods = new uint256[](1);
        allocStorage.vestingStartTimes = new uint256[](1);
        allocStorage.claimedSeconds = new uint256[](1);
        allocStorage.claimedFlows = new bool[](1);

        allocStorage.amounts[0] = 1000;
        allocStorage.vestingPeriods[0] = 100 days;
        allocStorage.vestingStartTimes[0] = block.timestamp;
        allocStorage.claimedSeconds[0] = 0;
        allocStorage.claimedFlows[0] = false;
        allocStorage.assignedPoolId = 0;
    }

    function splitTVS() external view returns (Allocation memory) {
        return _computeSplitArrays(allocStorage, 5000, 1);
    }

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
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / 10000;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
}
```

When we call `splitTVS()`:

```js
call
[call]from: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4to: Split.splitTVS()data: 0xdfa...e2401
call to Split.splitTVS errored: Error occurred: revert.

revert
	The transaction has been reverted to the initial state.
```

Details:
```js
{
	"error": "Failed to decode output: TypeError: overflow (argument=\"value\", value=35408467139433450592217433187231851964531694900788300625387963629091585785856, code=INVALID_ARGUMENT, version=6.14.0)"
}
```

### Mitigation

Before writing into the returned `Allocation memory alloc`, allocate each dynamic array with the appropriate length.
```js
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
        //@note initialize dynamic data
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        alloc.assignedPoolId = allocation.assignedPoolId;
    

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
  