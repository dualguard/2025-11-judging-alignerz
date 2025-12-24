# [000829] Missing loop increment in `calculateFeeAndNewAmountForOneTVS()`, causing the loop to never advance and never end
  
  ### Summary

The missing loop increment in `calculateFeeAndNewAmountForOneTVS()` will cause the function to run in an infinite loop, making all merge and split operations fail.


### Root Cause


In [FeesManager.sol:169-174](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the for loop has no increment statement, causing `i` to never increase:

```solidity
function calculateFeeAndNewAmountForOneTVS(...) returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // missing: unchecked { ++i; }
    }
}
```


### Internal Pre-conditions

Occur whenever user call `mergeTVS()` or `splitTVS()`

### External Pre-conditions

None

### Attack Path

1. User calls `splitTVS()` to split their vesting schedule with 1 flow
2. Function calls `calculateFeeAndNewAmountForOneTVS()` with `length = 1`
3. Loop starts with `i = 0` and condition `0 < 1` is true
4. Loop body executes but `i` is never incremented
5. Loop checks `i < length` again, still `0 < 1`, continues forever
6. Transaction runs out of gas and reverts

Same issue occurs with `mergeTVS()`.

### Impact

All merge and split operations fail with **out-of-gas** errors. Users cannot merge or split their vesting schedules, making these core protocol features completely unusable.


### PoC

_No response_

### Mitigation


Add the missing loop increment:

```diff
function calculateFeeAndNewAmountForOneTVS(...) returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
+       unchecked {   ++i; }
    }
}
```
  