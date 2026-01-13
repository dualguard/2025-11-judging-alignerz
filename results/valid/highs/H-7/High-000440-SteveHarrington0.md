# [000440] Forever Loop
  
  ### Summary

In `FeesManager :: calculateFeeAndNewAmountForOneTVS` there is a for loop which runs forever cause `i` never incremented.

### Root Cause

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

in the above snippet we can see that `i` is never incremented which result in forever loop and the tx will revert with out of gas error

### Internal Pre-conditions

Users have to call `mergeTVS` and `splitTVS` 

### External Pre-conditions

None

### Attack Path

Users will call either `mergeTVS` or `splitTVS` which triggers `calculateFeeAndNewAmountForOneTVS` and this function runs a for loop which will loop for ever and the tx will revert with `out of gas` error

### Impact

splitting TVS and merging TVS is not possible

### PoC

NA

### Mitigation

Increment `i` in the loop
  