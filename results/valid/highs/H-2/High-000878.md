# [000878] `calculateFeeAndNewAmountForOneTVS()` will always revert on first iteration due to unallocated array[/+] 
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` function returns a memory dynamic array:
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
```
The issue is that on the first iteration of the for loop we try to update the value at index 0, without earlier allocating the array:
```solidity
            newAmounts[i] = amounts[i] - feeAmount;
```
This will cause a revert each time on the first iteration. 

### Root Cause

`newAmounts` is left at the default zero-length, which makes it an uninitialized pointer. 

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

Allocate sufficient memory for the array using `new uint[](length)`.
  