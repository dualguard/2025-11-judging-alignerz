# [000328] Uninitialized Array  in `_computeSplitArrays` Function cause permenant DoS of SplitTVS
  
  ### Summary

The [_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132) helper function attempts to write to uninitialized dynamic arrays within a memory struct, causing all `splitTVS` operations to fail with a `Panic(0x32)` array-out-of-bounds error. The function declares an `Allocation memory alloc` return parameter but never initializes its array fields before accessing them. Since uninitialized dynamic arrays in memory have length zero, any write operation triggers an immediate panic revert, resulting in complete denial of service for the split functionality.

### Root Cause


In Solidity, when a struct containing dynamic arrays is declared in memory, the arrays are initialized with length zero by default. The developer must explicitly allocate memory for these arrays using the `new` keyword before they can be accessed.

The vulnerable code:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
---> ) internal view returns (Allocation memory alloc) {
   ...
    for (uint256 j; j < nbOfFlows;) {
  --->      alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;  // Panic(0x32)!
       ...
    }
}
```

**The Problem:**  when the code attempts `alloc.amounts[j] = ...`, it tries to access index `j` of an array with length 0. All five array fields (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) are uninitialized and have zero length, making any index access invalid.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. User calls `splitTVS(projectId, [5000, 5000], nftId)`
2. Function reaches `_computeSplitArrays(allocation, 5000, 1)`
3. Loop tries to access `alloc.amounts[0]`
4. Array has length 0
5. `Panic(0x32)` - transaction reverts


### Impact


- **Complete DoS**: `splitTVS` function is 100% non-functional
- **Universal Failure**: Affects every user attempting to split any NFT
- **Gas wastage**: All the gas is wasted


### PoC

_No response_

### Mitigation

Initialize all dynamic array fields before accessing them

  