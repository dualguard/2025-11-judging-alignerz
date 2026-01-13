# [000247] Uninitialized Memory Array Causes Out-of-Bounds Panic in Fee Calculation
  
  ### Summary

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function declares a return parameter `newAmounts` as a memory array, but never initializes it before attempting to write to it.

### Root Cause

The return parameter newAmounts is declared but never allocated:

```solidity
   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)  // ← Declared here
{
    // ← MISSING: newAmounts = new uint256[](length);
    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // Writing to length-0 array!
        unchecked { ++i; }
    }
}
```

### Internal Pre-conditions

1. User must own a vesting NFT
2. User must call either of the split or merge functions
3. The infinite loop bug earlier reported must be fixed (otherwise that bug manifests first)
4. The vesting allocation must have at least one token flow(i.e., `length > 0`)

### External Pre-conditions

None

### Attack Path

1. Infinite loop bug reported earlier on is fixed
2. Alice calls `splitTVS()`:

```solidity
   vesting.splitTVS(
    projectId: 0,
    percentages: [5000, 5000],  // 50/50 split
    splitNftId: 5
)
```

3. Detailed execution trace:

```
splitTVS() starts
  ↓
Retrieves allocation with 1 token flow
amounts = [10000 ether]
nbOfFlows = 1
  ↓
Calls: calculateFeeAndNewAmountForOneTVS(100, [10000 ether], 1)
  ↓
Inside calculateFeeAndNewAmountForOneTVS:
  ↓
Function signature:
returns (uint256 feeAmount, uint256[] memory newAmounts)
  ↓
At this point in memory:
- feeAmount = 0 (default uint256 value)
- newAmounts = [] (empty array, length = 0)
  ↓
Loop starts:
- i = 0
- Condition: 0 < 1 → TRUE
  ↓
Line 171: feeAmount += calculateFeeAmount(100, 10000 ether)
         feeAmount = 100 ether ✓
  ↓
Line 172: newAmounts[i] = amounts[i] - feeAmount
         Attempting: newAmounts[0] = 10000 ether - 100 ether
         
         BUT WAIT!
         ↓
         newAmounts.length = 0
         Trying to access index [0] of length-0 array
         ↓
         Solidity bounds check: Is 0 < 0? NO!
         ↓
         PANIC(0x32) - Array access out of bounds
  ↓
Transaction REVERTS immediately
```

4. Transaction reverts

### Impact

This bug is masked by the infinite loop bug reported earlier, and it results in split and merge functionalities failing.

### PoC

_No response_

### Mitigation

Initialize the `newAmounts` array before the loop.
  