# [000527] Cumulative fee accumulator in FeesManager::calculateFeeAndNewAmountForOneTVS leads to overcharging users and incorrect treasury transfers
  
  ### Summary

`FeesManager::calculateFeeAndNewAmountForOneTVS` uses a running total (`feeAmount`) when computing per-flow post-fee amounts, causing each subsequent flow to subtract the cumulative fee instead of its own fee. Callers such as `AlignerzVesting::mergeTVS` and `AlignerzVesting::splitTVS` consume the returned total fee and per-flow `newAmounts`, so the bug results in users being overcharged and the treasury receiving an incorrect total.

### Root Cause

Inside `FeesManager::calculateFeeAndNewAmountForOneTVS` the code accumulates the fee into `feeAmount` and then subtracts that accumulator from the current element:
`newAmounts[i] = amounts[i] - feeAmount;`
This subtracts prior flows' fees from the current flow instead of subtracting only the fee calculated for the current flow.

```javascript
  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
  @>>          newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Internal Pre-conditions

- The function `FeesManager::calculateFeeAndNewAmountForOneTVS` is called with `amounts` containing multiple flows (length > 1).
- Callers rely on both returned values: `feeAmount` (total fees to transfer to treasury) and `newAmounts` (per-flow net amounts stored back into storage).

### External Pre-conditions

- A user invokes `AlignerzVesting::mergeTVS` or `AlignerzVesting::splitTVS` where multiple flows exist on a TVS, causing `calculateFeeAndNewAmountForOneTVS` to be executed.
- No external guard prevents calls with typical multi-flow allocations.

### Attack Path

No response

### Impact

- Users are overcharged: flow 2 is reduced by fee1+fee2, flow 3 by fee1+fee2+fee3, etc. Net amounts stored on-chain are lower than intended.
- Treasury accounting is inconsistent with per-flow reductions; depending on caller logic the treasury may receive an incorrect sum (could be less or more than the intended per-flow fees depending on other caller-side code).

### PoC

No response

### Mitigation

```diff
-    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
-        for (uint256 i; i < length;) {
-            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-            newAmounts[i] = amounts[i] - feeAmount;
-        }
-    }
+    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+        require(length == amounts.length, "length mismatch");
+        newAmounts = new uint256[](length);
+        for (uint256 i = 0; i < length; ++i) {
+            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
+            feeAmount += fee;
+            newAmounts[i] = amounts[i] - fee;
+        }
+    }
```
  