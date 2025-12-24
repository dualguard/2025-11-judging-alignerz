# [000964] Incorrect comparison operator in `getUnclaimedAmounts`
  
  ### Summary

The `getUnclaimedAmounts` function uses an incorrect comparison operator: when the TVS token matches the target token. The function returns 0 instead of calculating unclaimed tokens. This results in zero `totalUnclaimedAmounts`, which breaks the dividend distribution: a revert is possible when dividing by zero, or an incorrect distribution. Holders of valid TVS do not receive dividends.

### Root Cause

The `getUnclaimedAmounts` function uses an incorrect comparison operator:

```solidity 
 function getUnclaimedAmounts(
        uint256 nftId
    ) public returns (uint256 amount) {
>>>     if (address(token) == address(vesting.allocationOf(nftId).token))
            return 0; 
```

The documentation shows that this is an incorrect implementation of the comparison: 
“5% of our profit will be redistributed directly to our TVS holders, proportionally to their locked tokens.”

“Dividends are distributed only to TVS with the $A26Z token.”

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user holds TVS tokens and wants to receive dividends.
2. Calls this function to distribute undistributed tokens in order to receive their share of dividends. 

### Impact

Users are completely deprived of dividends.

### PoC

_No response_

### Mitigation

_No response_
  