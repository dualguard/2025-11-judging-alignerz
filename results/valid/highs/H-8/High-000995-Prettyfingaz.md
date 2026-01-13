# [000995] Infinite loop in `A26ZDividendDistributor::getUnclaimedAmounts`
  
  ### Summary

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
...SNIP...
        //@audit bug infinite loop here if claimedFlows
        //@audit we forget to increment this 
        for (uint i; i < len;) {
  @>>       if (claimedFlows[i]) /** ++i */ continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

the `getUnclaimedAmounts` function has a continue statement in the for loop highlighed above. However, the counter is not incremented before the continue statement is hit. This will allow the if block to run in an endless loop if `claimedSeconds[i]` at any index i is true.


This will DOS all functions that trigger `_setAmount` as the logic within the function routes its way into `getUnclaimedAmounts`. 

Functions like `setUpTheDividends` and `setAmounts` will be bricked.

### Root Cause

Late counter update in `getUnclaimedAmounts`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

read summary

### Impact

DOS of core functionalities

### PoC

_No response_

### Mitigation

Allow the counter to be updated earlier withing the for statement declaration
  