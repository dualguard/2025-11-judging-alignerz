# [000965] incorrect calculation of fees in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

In the `calculateFeeAndNewAmountForOneTVS` function, the accumulated amount of fees is deducted from each stream instead of just the fee for that stream. This leads to a loss of funds for users: the more streams and their amounts, the greater the loss. The vulnerability affects `splitTVS` and `mergeTVS` operations.

### Root Cause

The function loop uses the accumulated variable `feeAmount` to deduct from each stream. 
```solidity 
    function calculateFeeAndNewAmountForOneTVS(
...
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        uint256[] memory newAmounts = new uint256[](length);
        for (uint256 i; i < length; i++) {

>>>     feeAmount += calculateFeeAmount(feeRate, amounts[i]); 
            newAmounts[i] = amounts[i] - feeAmount; 
        }
        return (feeAmount, newAmounts);
    }
```
The longer the length, the higher the commission users will pay. This is because each iteration increases the same variable `feeAmount`. 

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user calls the mergeTVS or splitTVS function
2. Passes a TVS array with a length > 1

### Impact

Users directly lose their funds

### PoC

Let's consider the following PoC. We take two identical numbers. Accordingly, the new array will also contain two reduced identical numbers, but in reality this is not the case. 
```solidity 
 function test_wrongFeeCalculation() public {
        uint256 feeRate = 1000;
        uint256 amount1 = 1000;
        uint256 amount2 = 1000;
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = amount1;
        amounts[1] = amount2;
        (uint256 feeAmount, uint256[] memory newAmounts) = vesting
            .calculateFeeAndNewAmountForOneTVS(
                feeRate,
                amounts,
                amounts.length
            );
        assertEq(newAmounts[0], 900);
        assertEq(newAmounts[1], 800);
    }
```

### Mitigation

_No response_
  