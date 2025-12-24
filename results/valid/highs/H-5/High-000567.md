# [000567] Cumulative Fee Accumulation Bug in Fee Calculation Function Causes Incorrect Flow Amount Deductions and Potential Underflow Reverts
  
  

## Summary

The incorrect accumulation of fees across multiple flows in `calculateFeeAndNewAmountForOneTVS()` will cause excessive fee deductions and potential transaction reverts for users performing split or merge operations, as each flow amount is reduced by the cumulative sum of all previous fees instead of its own individual fee.

```js
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); // <-- @audit
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
&nbsp;

## Root Cause

In [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the `calculateFeeAndNewAmountForOneTVS()` function accumulates the fee amount across all flows in the loop before subtracting it from each individual flow amount. Specifically, on line 171, `feeAmount` is incremented with the fee for the current flow, and then on line 172, this cumulative `feeAmount` (which includes fees from all previous flows) is subtracted from `amounts[i]` to calculate `newAmounts[i]`. This causes each subsequent flow to be charged not only its own fee but also the sum of all previous fees, resulting in progressively larger incorrect deductions. Additionally, this can cause an underflow revert when the cumulative `feeAmount` exceeds `amounts[i]` for any flow in the sequence.

&nbsp;

## Internal Pre-conditions

1. A user needs to call `splitTVS()` or `mergeTVS()` with a TVS that contains multiple flows (at least 2 flows)
2. The `splitFeeRate` or `mergeFeeRate` needs to be set to a value greater than 0 basis points
3. The sum of cumulative fees from previous flows needs to be less than or equal to the current flow amount to avoid an underflow revert (for the bug to manifest without reverting)

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. A user calls `splitTVS()` or `mergeTVS()` with a TVS containing multiple flows
2. The function calls `calculateFeeAndNewAmountForOneTVS()` to calculate fees and new amounts for all flows
3. In the first iteration (i=0), the function calculates the fee for `amounts[0]` and correctly sets `newAmounts[0] = amounts[0] - feeAmount`
4. In the second iteration (i=1), the function adds the fee for `amounts[1]` to the cumulative `feeAmount`, then incorrectly sets `newAmounts[1] = amounts[1] - feeAmount`, where `feeAmount` now includes fees from both flow 0 and flow 1
5. For each subsequent iteration, the cumulative `feeAmount` continues to grow, causing each flow to be charged an increasingly larger incorrect fee
6. If at any point the cumulative `feeAmount` exceeds `amounts[i]`, the subtraction will underflow and the transaction will revert

&nbsp;

## Impact

Users performing split or merge operations suffer incorrect fee deductions that increase with each flow in the TVS. The first flow is charged correctly, but each subsequent flow is overcharged by the sum of all previous fees. This can result in significant loss of funds proportional to the number of flows and the fee rate. Additionally, transactions may revert due to underflow when the cumulative fee exceeds a flow's amount, preventing users from completing split or merge operations on TVSs with many flows or high fee rates.

&nbsp;

## Proof of Concept

N/A


&nbsp;

## Mitigation

Calculate the individual fee for each flow before accumulating it into the total fee amount, and subtract only the individual fee from each flow amount:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    uint256 feeAmountForFlow;
    for (uint256 i; i < length;) {
+        feeAmountForFlow = calculateFeeAmount(feeRate, amounts[i]);
+        feeAmount += feeAmountForFlow;
+        newAmounts[i] = amounts[i] - feeAmountForFlow;
        unchecked {
            ++i;
        }
    }
}
```
  