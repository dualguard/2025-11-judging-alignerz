# [000775] The missing iterator increment in the fee calculation loop will cause an infinite loop
  
  ### Summary

The missing iterator increment in the fee calculation loop will cause an infinite loop and gas exhaustion for all users of `mergeTVS` and `splitTVS` if the helper is ever used with a non-empty array and its array write no longer reverts, effectively creating a full denial of service on these operations.

### Root Cause

In `FeesManager.sol:L169-L173` the loop inside `calculateFeeAndNewAmountForOneTVS` never increments `i`, so the condition `i < length` is always true once entered and the function cannot terminate when `length > 0` and the body does not revert for another reason.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
>>  for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

This is currently hidden by the uninitialized `newAmounts` bug (which is subject of another submission), but as soon as the array write starts to succeed for `i = 0` and `length > 0`, the function will loop forever because `i` never changes.

### Internal Pre-conditions

1. The helper `calculateFeeAndNewAmountForOneTVS` is called with `length > 0` and `amounts.length >= length`, for example via `mergeTVS` or `splitTVS` on a TVS that has at least one flow.  


### External Pre-conditions

1. No specific conditions are needed; standard ERC20 behavior and a normal vesting setup are enough to reach the buggy loop once the array write succeeds.

### Attack Path

1. The admin deploys or upgrades the contract so that `calculateFeeAndNewAmountForOneTVS` is used with a correctly allocated `newAmounts`
2. A user obtains a TVS NFT with `amounts.length > 0` through the normal flows and calls `splitTVS` or `mergeTVS`.  
3. `AlignerzVesting.mergeTVS` or `splitTVS` calls `calculateFeeAndNewAmountForOneTVS` with `length = nbOfFlows > 0`.  
4. Inside the helper, `i` is initialized to zero and the loop condition `i < length` is true, so the body executes and writes `newAmounts[0] = ...` successfully.  
5. At the end of the iteration, `i` is never incremented, so `i` remains zero; the loop condition is still true and the function repeats the same body forever until it runs out of gas, causing the user’s `mergeTVS` or `splitTVS` transaction to always revert with an out-of-gas style failure.

### Impact

All `mergeTVS` and `splitTVS` calls will run into an infinite loop and revert due to gas exhaustion, fully blocking users from merging or splitting TVSs, with no direct fund theft but a complete denial of service on these features.

### PoC

Because the uninitialized `newAmounts` bug currently makes the function revert earlier with an array out-of-bounds panic, the missing iterator increment does not change observable behavior in the deployed code today. However, if you apply only the array-allocation fix from the first issue (allocating `newAmounts = new uint256[](length)` and leaving the loop header as `for (uint256 i; i < length;)` without `++i`), then re-running the same user flow tests in `test/FeesManagerBug.t.sol` (reward project setup, claim TVS NFT, then calling `mergeTVS` or `splitTVS`) would make those tests hang and revert due to an infinite loop in `calculateFeeAndNewAmountForOneTVS`, showing this issue in practice.

### Mitigation

Rewrite the function as;
```diff
-    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
-        for (uint256 i; i < length;) {
-            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-            newAmounts[i] = amounts[i] - feeAmount;
-        }
-    }
+    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+        for (uint256 i; i < length;) {
+            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
+            newAmounts[i] = amounts[i] - feeAmount;
+            unchecked {
+                ++i;
+            }
+        }
+    }
```
  