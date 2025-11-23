# [000483] Missing memory allocation for `newAmounts` array causes revert
  
  ### Summary

The `FeeManager::calculateFeeAndNewAmountForOneTVS()` function writes to `newAmounts[i]` without allocating memory for the array, causing an immediate revert whenever the loop executes.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Root Cause

`newAmounts` is declared but never initialized with `new uint256[](length)`, leaving it without a valid memory reference when indexed leading to out of bound revert.

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

No attack path. Direct issue.

### Impact

The function always reverts for any length > 0.

The `mergeTVS()` & `splitTVS()` function rely on this function . If this function reverts , then `mergeTVS` & `splitTVS()` would also revert.

### PoC

NA

### Mitigation

Initialize the memory array before assigning:

```diff
 function calculateFeeAndNewAmountForOneTVS(..){
+     newAmounts = new uint256[](length);
       ...
       ...
}
```
  