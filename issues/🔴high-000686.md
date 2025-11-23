# [000686] Uninitialized memory arrays in `_computeSplitArrays` cause immediate transaction revert, rendering the split functionality unusable
  
  ### Summary

The `_computeSplitArrays` function declares a `memory` struct `alloc` but fails to initialize its internal dynamic arrays using the `new` keyword. In Solidity, dynamic arrays in memory are initialized with a length of 0 by default. As a result, any attempt to assign values to indices within these arrays triggers an "Array Index Out of Bounds" panic (0x32), causing the transaction to revert. This renders the `splitTVS` function completely inoperable.

### Root Cause


In `AlignerzVesting.sol`, inside the `_computeSplitArrays` function, the `Allocation memory alloc` struct is declared but its array members are never allocated memory.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141
```solidity
// AlignerzVesting.sol

function _computeSplitArrays(...) internal view returns (Allocation memory alloc) {
    // ...
    // ROOT CAUSE: alloc.amounts is a memory array with length 0 by default.
    // No 'new uint256[](size)' is called here.

    for (uint256 j; j < nbOfFlows;) {
        // VULNERABILITY: Writing to index 'j' of a length-0 array causes a revert.
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        // ...
    }
}
```

The loop attempts to write to `alloc.amounts[j]`, but since the array has zero length, this operation is invalid and causes the EVM to throw an exception.

### Internal Pre-conditions

1.  A user attempts to call `splitTVS` with a valid NFT ID and percentage array.
2.  The NFT has at least one vesting flow (`nbOfFlows > 0`), which is always true for a valid TVS.

### External Pre-conditions

_No response_

### Attack Path

1.  A user calls `splitTVS` to split their vesting schedule.
2.  The contract calls `_computeSplitArrays`.
3.  The function enters the loop and attempts to execute `alloc.amounts[0] = ...`.
4.  The EVM detects an out-of-bounds access because `alloc.amounts.length` is 0.
5.  The transaction reverts with Panic Code 0x32.

### Impact

The `splitTVS` feature is completely broken and cannot be used by any user. 

### PoC

_No response_

### Mitigation

Explicitly initialize all dynamic arrays within the `alloc` struct using the `new` keyword before entering the loop.
  