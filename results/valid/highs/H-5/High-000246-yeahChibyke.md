# [000246] Cumulative Fee Subtraction causes Users to be Overcharged in Multi-Flow Fee Calculation
  
  ### Summary

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function contains a logic error in how it calculates fees for allocations with multiple token flows. While the function correctly accumulates the total fee amount across all flows, it incorrectly subtracts the cumulative total from each flow instead of subtracting only that individual flow's fee. This causes later flows in the array to have increasingly excessive fees deducted, resulting in users receiving less tokens than they should after fee deduction.

### Root Cause

The function calculates fees correctly, but applies them incorrectly:

```solidity
   function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // ✓ Accumulates total
        newAmounts[i] = amounts[i] - feeAmount;  // Subtracts cumulative total!
        unchecked { ++i; }
    }
}
```

The `feeAmount` variable serves dual purposes but only one is implemented correctly:

1. Track total fees (for returning to caller) - works correctly
2. Individual flow fee (for calculating newAmount) - uses wrong value

### Internal Pre-conditions

1. User must own a vesting NFT with multiple token flows (i.e., `allocation.amount.length >= 2)`). This is because single-flow allocations are unaffected bug as first iteration is correct, but bug severity increases with number of flows
2. User must call either of the split or merge functions to trigger fee calculation
3. The earlier reported bugs (infinite loop and uninitialized array) must be fixed, otherwise those bugs manifest first and mask this one

### External Pre-conditions

None

### Attack Path

**Scenario:** 

Alice has an NFT with 3 token flows and wants to split it. The other bugs have been fixed.

**Setup:**

  Alice's NFT allocation:

  - Flow 0: 1,000 tokens vesting over 90 days
  - Flow 1: 2,000 tokens vesting over 100 days
  - Flow 2: 3,000 tokens vesting over 365 days
  - Total: 6,000 tokens across 3 flows
  
 Protocol configuration: `splitFeeRate = 100` (1% fee = 100 basis points out 10,000)

**Expected Behaviour (Correct Fee Calculation):**

```
Flow 0: 
  Fee = 1,000 × 100 / 10,000 = 10 tokens
  After fee = 1,000 - 10 = 990 tokens ✓

Flow 1:
  Fee = 2,000 × 100 / 10,000 = 20 tokens  
  After fee = 2,000 - 20 = 1,980 tokens ✓

Flow 2:
  Fee = 3,000 × 100 / 10,000 = 30 tokens
  After fee = 3,000 - 30 = 2,970 tokens ✓

Total fees collected: 10 + 20 + 30 = 60 tokens
Alice receives: 990 + 1,980 + 2,970 = 5,940 tokens
```

**Actual Behaviour (With Bug):**

```
Iteration 0 (Flow 0):
  flowFee = 1,000 × 100 / 10,000 = 10 tokens
  feeAmount = 0 + 10 = 10 tokens (cumulative)
  newAmounts[0] = 1,000 - 10 = 990 tokens ✓ CORRECT
  
Iteration 1 (Flow 1):
  flowFee = 2,000 × 100 / 10,000 = 20 tokens
  feeAmount = 10 + 20 = 30 tokens (cumulative)
  newAmounts[1] = 2,000 - 30 = 1,970 tokens ❌ WRONG!
  (Should be 2,000 - 20 = 1,980 tokens)
  Loss to user: 10 tokens
  
Iteration 2 (Flow 2):
  flowFee = 3,000 × 100 / 10,000 = 30 tokens
  feeAmount = 30 + 30 = 60 tokens (cumulative)
  newAmounts[2] = 3,000 - 60 = 2,940 tokens ❌ WRONG!
  (Should be 3,000 - 30 = 2,970 tokens)
  Loss to user: 30 tokens

Total fees "collected": 60 tokens (matches expected - tracking is correct)
Alice receives: 990 + 1,970 + 2,940 = 5,900 tokens ❌
Alice should receive: 5,940 tokens ✓

Alice's loss: 40 tokens
```

### Impact

Tokens end up missing:

- Total before fee: 6,000 tokens
- Total fee collected: 60 tokens
- Expected total after fee: 5,940 tokens
- Actual total after fee: 5,900 tokens
- 40 tokens end up missing

Users end up getting overcharged on splits and fees, and because this bug is overshadowed by earlier bugs (infinite loop and uninitialized array), protocol never realizes this.

### PoC

_No response_

### Mitigation

Use a local variable to store the individual flow's fee, separate from the cumulative total:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);  // Fix uninitialized array bug
    
    for (uint256 i; i < length;) {
-       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-       newAmounts[i] = amounts[i] - feeAmount;
+       uint256 flowFee = calculateFeeAmount(feeRate, amounts[i]);
+       feeAmount += flowFee;  // Accumulate for total
+       newAmounts[i] = amounts[i] - flowFee;  // Subtract only this flow's fee
        unchecked {
            ++i;  // Fix infinite loop bug
        }
    }
}
```
  