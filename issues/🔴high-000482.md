# [000482] Incorrect fee calculation logic inside the FeeManager::calculateFeeAndNewAmountForOneTVS() function
  
  ### Summary

The function subtracts the cumulative fee (feeAmount) from each iteration instead of subtracting only the fee belonging to that specific amount, resulting in increased fee deductions .

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174

### Root Cause

The loop aggregates fees into `feeAmount` and then applies this total accumulated value to every subsequent iteration. This causes the `newAmounts` to be reduced by all previous fees rather than its own individual fee, leading to incorrect and inflated fee deductions.

```solidity
 function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]); //<= here
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

Logic issue. No attack path.

### Impact

Wrong fee gets deducted with each iteration. This function is called inside the mergeTVS() and splitTVS() , so splitTVS/mergeTVS funcitons over charge later flows (users lose tokens) and create an accounting gap where tokens become unallocated .

### PoC

Na

### Mitigation

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
-            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-            newAmounts[i] = amounts[i] - feeAmount;
+           uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
+           newAmounts[i] = amounts[i] - fee;
+           feeAmount += fee;    
      }
    }
```
  