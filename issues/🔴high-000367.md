# [000367] H11 Cumulative fee subtraction in FeesManager causes loss of funds
  
  ## Summary

The `FeesManager` contract contains a critical logic bug in `calculateFeeAndNewAmountForOneTVS`. It calculates fees cumulatively, subtracting the sum of all previous fees from the current flow's amount. This results in severe overcharging for users with multiple vesting flows, potentially wiping out their allocations.

## Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
1. @>       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
2. @>       newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

At marker 1, `feeAmount` accumulates the fee for *each* flow.
At marker 2, this *cumulative* `feeAmount` is subtracted from `amounts[i]`.

**Example Scenario:**
*   User has 3 flows of 100 tokens each. Fee is 1%.
*   **Iteration 0**: `feeAmount` = 1. `newAmounts[0]` = 100 - 1 = 99. (Correct)
*   **Iteration 1**: `feeAmount` = 1 + 1 = 2. `newAmounts[1]` = 100 - 2 = 98. (Incorrect, should be 99)
*   **Iteration 2**: `feeAmount` = 2 + 1 = 3. `newAmounts[2]` = 100 - 3 = 97. (Incorrect, should be 99)

Users are charged the fees of all previous flows again for every subsequent flow. For a large number of flows, this can exceed the flow amount, causing an underflow revert or zero allocation.

## Internal Pre-Conditions

1.  `splitFeeRate` or `mergeFeeRate` must be non-zero.
2.  The allocation must have multiple flows (`length > 1`).

## External Pre-Conditions

None.

## Attack Path

1.  User calls `splitTVS` or `mergeTVS` on an NFT with multiple vesting flows.
2.  The fee calculation logic progressively deducts more than the required fee from later flows.
3.  User loses funds or the transaction reverts if `feeAmount > amounts[i]`.

## Impact

**High**. Loss of funds for users. Users with many vesting flows will be significantly penalized.

## PoC

Trivial.

## Mitigation

Use a local variable for the current fee, not the cumulative total, when calculating the new amount.

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
-           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-           newAmounts[i] = amounts[i] - feeAmount;
+           uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
+           newAmounts[i] = amounts[i] - currentFee;
+           feeAmount += currentFee;
        }
    }
```

  