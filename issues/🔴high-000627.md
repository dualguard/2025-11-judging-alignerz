# [000627] Users will receive incorrect TVS amounts due to cumulative fee subtraction in `calculateFeeAndNewAmountForOneTVS` function
  
  ### Summary

A faulty fee calculation in `calculateFeeAndNewAmountForOneTVS()` will cause incorrect deductions for users, as the function subtracts a cumulative fee total instead of the per-flow fee, resulting in progressively larger and inaccurate reductions across all vesting amounts.

### Root Cause

In [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171) function
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
@>            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
where `feeAmount` is the running total of all previous fees, instead of subtracting only the current fee for the corresponding amount.
This compounds fees across flows and produces incorrect user balances.

### Internal Pre-conditions

A user merge or split nfts, having multiple allocation amounts

### External Pre-conditions

_No response_

### Attack Path

For amounts [100, 200, 300] at a 1% fee rate:
fee1 = 1
fee2 = 2
fee3 = 3
Instead of deducting each fee individually, the function subtracts the growing cumulative total:
Amount1: 100 − 1 = 99
Amount2: 200 − (1 + 2) = 197 (incorrect, should be 198)
Amount3: 300 − (1 + 2 + 3) = 294 (incorrect, should be 297)


### Impact

Users lose funds due to over-charged fees

### PoC

_No response_

### Mitigation

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
-            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-            newAmounts[i] = amounts[i] - feeAmount;
+           uint256 amount = amounts[i];
+           uint256 fee = calculateFeeAmount(feeRate, amount);

+           feeAmount += fee;
+           newAmounts[i] = amount - fee;
        }
    }
```          
  