# [000879] `calculateFeeAndNewAmountForOneTVS()` will always OOG due to lack of iterator incremenet
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` function contains a for loop that never increments the iterator `i`:
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
This will permanently break all functionalities that use this function, causing OOG on each call. 

### Root Cause

Lack of `i++/++i` in the for loop.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

`mergeTVS()` and `splitTVS()` functionalities are permanently broken.

### PoC

-

### Mitigation

Increment the iterator properly.
  