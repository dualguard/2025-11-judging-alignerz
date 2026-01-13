# [000477] FeesManager::calculateFeeAndNewAmountForOneTVS Incorrect Fee Deduction Causes Progressive Over-Subtraction and User Funds Loss
  
  ### Summary

FeesManager::calculateFeeAndNewAmountForOneTVS uses a cumulative feeAmount when subtracting from each element of amounts.

This results in progressively increasing deductions and causes users to lose funds beyond the intended single-TVS fee rate.

### Root Cause


feeAmount is cumulative, but the code subtracts the total cumulative fee from each individual amounts[i].

Each step subtracts all previous fees + current fee, inflating losses.

```solidity
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.	User calls mergeTVS() or splitTVS().
2.	FeesManager calculates fees using cumulative subtraction.
3.	User receives an incorrect (lower) amount.
4.	Loss compounds with more TVSs and more operations.

### Impact

Direct loss of user funds through inflated fee deductions

### PoC

The PoC below does not use the actual FeesManager contract because that contract currently has two other unrelated bugs (missing allocation and missing i++) which cause it to always revert.
Instead, the test re-implements the same fee logic in isolation to demonstrate the overcharging behavior:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

contract FeesManagerCumulativeFeeBugPoC is Test {
    uint256 constant public BASIS_POINT = 10_000;
    
    function test_CumulativeFeeBug() public {
        uint256[] memory amounts = new uint256[](4);
        amounts[0] = 1000;
        amounts[1] = 2000;
        amounts[2] = 3000;
        amounts[3] = 4000;
        uint256 feeRate = 100; // 1% fee
        
        (uint256 totalFee, uint256[] memory newAmounts) = 
            calculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);
        
        for (uint i = 0; i < amounts.length; i++) {
            uint256 correctResult = amounts[i] - calculateFeeAmount(feeRate, amounts[i]);
            uint256 userLoss = correctResult - newAmounts[i];
            console.log("[%d] Loss: %d", i, userLoss);
        }
        
        uint256 correctResult1 = amounts[1] - calculateFeeAmount(feeRate, amounts[1]);
        assertTrue(newAmounts[1] != correctResult1, "Bug confirmed");
    }

    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            i++;
        }
    }
    
    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256) {
        return amount * feeRate / BASIS_POINT;
    }
}
```

```
forge test --mt test_CumulativeFeeBug  -vv
```

### Mitigation

```solidity
for (uint256 i; i < length; i++) {
    uint256 thisFee = calculateFeeAmount(feeRate, amounts[i]);
    feeAmount += thisFee;                 // total fee over all TVSs
    newAmounts[i] = amounts[i] - thisFee; // only this TVS’s fee
}
```
  