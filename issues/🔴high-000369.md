# [000369] H09 Missing loop increment in FeesManager causes infinite loop and underflow
  
  ## Summary

The `FeesManager` contract contains a critical bug in `calculateFeeAndNewAmountForOneTVS` where the `for` loop is missing an increment statement for the loop counter `i`. This results in an infinite loop where the fee is repeatedly accumulated until it exceeds the flow amount, causing an arithmetic underflow revert.

## Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
1. @>   for (uint256 i; i < length; ) { 
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

At marker 1, the loop condition `i < length` is checked, but `i` is never incremented within the loop body.
1.  The loop runs indefinitely for `i = 0`.
2.  `feeAmount` accumulates `calculateFeeAmount(feeRate, amounts[0])` in every iteration.
3.  Eventually, `feeAmount` becomes greater than `amounts[0]`.
4.  The subtraction `amounts[i] - feeAmount` triggers an **Arithmetic Underflow** (Panic 0x11).

## Internal Pre-Conditions

1.  `length > 0`.
2.  `feeRate > 0` (to cause underflow) or `feeRate == 0` (infinite loop leading to Out of Gas).

## External Pre-Conditions

None.

## Attack Path

1.  User calls `splitTVS` or `mergeTVS`.
2.  `calculateFeeAndNewAmountForOneTVS` is called.
3.  The transaction reverts due to Underflow or Out of Gas.

## Impact

**High**. `splitTVS` and `mergeTVS` are completely unusable.

## PoC

Trivial

## Mitigation

Add the increment statement to the loop.

```diff
-       for (uint256 i; i < length; ) {
+       for (uint256 i; i < length; ++i) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
```

  