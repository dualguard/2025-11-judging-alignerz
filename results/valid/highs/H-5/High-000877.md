# [000877] `calculateFeeAndNewAmountForOneTVS()` fee math is wrong
  
  ### Summary

In `calculateFeeAndNewAmountForOneTVS()`, we subtract the cumulative `feeAmount` from all flows from `amount[i]` on each iteration:
```solidity
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
```
This breaks the function for any call with `length > 1`.

### Root Cause

Using cumulative fee over all flows instead of the fee for given iteration.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

`mergeTVS()` and `splitTVS()` are broken for all calls where `lenght > 1`.

### PoC

-

### Mitigation

Use the following code instead:
```solidity
uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
feeAmount += fee;
newAmounts[i] = amounts[i] - fee;
```
  