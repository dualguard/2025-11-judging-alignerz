# [001024] Skipped increment expression in `for` loop leads to infinite loop and Divident Distributor disruption
  
  ### Summary

`A26ZDividendDistributor::getUnclaimedAmounts()` is missing increment iterator in 2 different branches and will render an infinite loop. This blocks admin to set key storage variables and dividend distribution. 

### Root Cause

In [getUnclaimedAmounts()](https://github.com/dualguard/2025-11-alignerz-AlexCZM/blob/1a33c2cc49b18ac319688bacc012964913d42253/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L159) function 
there's the following for loop:

```solidity
// getUnclaimedAmounts() 
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue;                           // @audit skipped ++i
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;                                            // @audit skipped ++i
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
```

In case `claimedFlows[i]` is true or `claimedSeconds[i]` is `0`, the `continue` ends the current iteration; increment expression, `++i` is skipped and the for loop is executed again with the same `i` value, resulting an infinite loop. 

### Internal Pre-conditions

- at least one claimed 'flow': claimedFlows[i] == true  OR
- a flow to now have been claimed: claimedSeconds[i] == 0

### External Pre-conditions

None

### Attack Path

Admin calls `setUpTheDividends()` or `setAmounts()`. Both functions calls in the end the `getUnclaimedAmounts()` function, which is the vulnerable function. 
If one of the internal pre-condition is present, the for loop  becomes an infinite loop. 

### Impact

Admin can't set the dividends nor the total unclaimed amounts for the next period. 
The Dividend distribution mechanism is blocked. 

### PoC

_No response_

### Mitigation

Consider removing the `unckecked` block from `getUnclaimedAmounts` and move the increment in the for header:
In this way the counter iterator will be incremented each time. 
```diff
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
//...
-        for (uint i; i < len; ) {
+        for (uint i; i < len; i++) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
-            unchecked {
-                ++i;
-            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```
  