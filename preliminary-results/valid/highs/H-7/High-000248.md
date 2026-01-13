# [000248] Missing Loop Increment Causes Infinite Loop and Gas Exhaustion in Fee Calculation
  
  ### Summary

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function contains a for-loop that lacks an increment statement and this causes the loop counter `i` to remain at 0 indefinitely. This results in an infinite loop that will exhaust all available gas and revert any transaction that calls this function.

Note: this function also suffers from an uninitialized return array and a cumulative-fee logic bug discussed in H-4 and H-5 (I have also reported them below - judge please take note); the observed failure mode may depend on execution order and which bug triggers first. Apply a combined fix that addresses loop increment, array initialization, and per-flow fee computation.

### Root Cause

The for loop is missing the increment statement that advances the loop counter:

```solidity
   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
      public
      pure
      returns (uint256 feeAmount, uint256[] memory newAmounts)
   {
      for (uint256 i; i < length;) {
         feeAmount += calculateFeeAmount(feeRate, amounts[i]);
         newAmounts[i] = amounts[i] - feeAmount;
         // ← MISSING: unchecked { ++i; }
      }
   }
```

**The Problem:**

- loop initializes with `i = 0`
- loop conditions is `i < length`
- loop body executes operations with `i = 0`
- loop counter `i` is never incremented
- control returns to check condition: `0 < length` is still `TRUE`
- loop executes with `i = 0`
- this continues until gas runs out

### Internal Pre-conditions

1. User must own a vesting NFT
2. User must attempt to call either `AlignerzVesting::splitTVS()` to split their NFT into multiple parts, or `AlignerzVesting::mergeTVS()` to combine multiple NFTs into one
3. `splitFeeRate` or `mergeFeeRate` must be set to any value > 0 so as to trigger fee calculation

### External Pre-conditions

None

### Attack Path

1. Alice owns an NFT which represents 10,000 tokens vesting over 365 days, and she wants to keep 60% and gift/sell 40% to Bob
2. Protocol has split fee configured: `splitFeeRate = 100; // 1% fee (100 basis points out of 10,000)
3. Alice calls `::splitTVS()`:

```solidity
   vesting.splitTVS(
      projectId: 0,
      percentages: [6000, 4000],  // 60% and 40%
      splitNftId: 5
   )
```
4. Execution trace:

```
   splitTVS() called
  ↓
Line 1121: Retrieves allocation.amounts (e.g., [10000 ether])
  ↓
Line 1122: Sets nbOfFlows = 1
  ↓
Line 1123: Calls calculateFeeAndNewAmountForOneTVS(100, [10000 ether], 1)
  ↓
Inside calculateFeeAndNewAmountForOneTVS:
  ↓
Line 170: Loop starts with i = 0, length = 1
  ↓
Line 171: Condition check: 0 < 1 → TRUE, enter loop
  ↓
Line 172: feeAmount += calculateFeeAmount(100, 10000 ether)
          feeAmount = 0 + 100 ether = 100 ether
  ↓
Line 173: newAmounts[0] = 10000 ether - 100 ether = 9900 ether
          (NOTE: This would fail with uninitialized array bug, but let's assume it doesn't)
  ↓
Line 174: End of loop body... NO INCREMENT!
  ↓
Loop goes back to condition check:
  ↓
Line 171: Condition check: 0 < 1 → STILL TRUE (i is still 0!)
  ↓
Line 172: feeAmount += calculateFeeAmount(100, 10000 ether)
          feeAmount = 100 ether + 100 ether = 200 ether
  ↓
Line 173: newAmounts[0] = 10000 ether - 200 ether = 9800 ether
  ↓
Loop continues... i is STILL 0
  ↓
This repeats indefinitely:
- Iteration 3: feeAmount = 300 ether
- Iteration 4: feeAmount = 400 ether
- Iteration 5: feeAmount = 500 ether
- ... continues ...
- Eventually: Out of gas!
  ↓
Transaction REVERTS with "out of gas"
```

5. ALice's transaction fails

### Impact

Split and merge functionalities completely broken

### PoC

_No response_

### Mitigation

Add the missing increment statement to the for loop.

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
+   newAmounts = new uint256[](length);  // Also fix uninitialized array bug
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
+       unchecked {
+           ++i;
+       }
    }
}
```
  