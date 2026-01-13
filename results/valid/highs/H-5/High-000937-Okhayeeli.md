# [000937] Fee subtraction in `calculateFeeAndNewAmountForOneTVS` causes incorrect fee calculation and user fund loss
  
  ### Summary

The  fee subtraction logic in `FeesManager::calculateFeeAndNewAmountForOneTVS` will cause excessive fee deduction for users as itaccumulates total fees across all token flows but subtracts this cumulative total from each individual flow amount, resulting in users paying more fees  than intended.

### Root Cause

In [FeesManager.sol:171-172](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L127) assuming `(the i++ is present)` , the `feeAmount`  accumulates fees across all iterations `(feeAmount += calculateFeeAmount(...))`, but then this cumulative total is subtracted from each individual amount `(newAmounts[i] = amounts[i] - feeAmount)`. This means the first flow gets the correct fee deduction, but subsequent flows get increasingly larger deductions as `feeAmount` grows.

### Impact

users suffer excess fee deduction

### PoC
```solidity
function test_CumulativeFeeSubtraction() public {  
    // Setup: TVS with 3 flows of 1000 tokens each  
    uint256[] memory amounts = new uint256[](3);  
    amounts[0] = 1000e18;  
    amounts[1] = 1000e18;  
    amounts[2] = 1000e18;  
      
    // 1% fee (100 basis points)  
    uint256 feeRate = 100;  
      
    (uint256 totalFee, uint256[] memory newAmounts) =   
        feesManager.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 3);  
      
    // Expected: Each flow loses 10 tokens (1%), total fee = 30 tokens  
    // Actual: Flow 0 = 990, Flow 1 = 980, Flow 2 = 970, total fee = 30  
    // User gets: 990 + 980 + 970 = 2940 instead of 2970  
    // User loses extra 30 tokens beyond the intended 30 token fee  
      
    assertEq(newAmounts[0], 990e18); 
    assertEq(newAmounts[1], 980e18); 
    assertEq(newAmounts[2], 970e18); 
}
```

### Mitigation

Calculate individual fee per flow instead of cumulative:
  