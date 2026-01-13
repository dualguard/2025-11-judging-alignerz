# [001021] Missing increment from for loop leads to splitTVS() and mergeTVS() DOS
  
  ### Summary

The missing for loop increment from  `FeesManager::calculateFeeAndNewAmountForOneTVS()` leads to infinite loop and `mergeTVS()` and `splitTVS()` DOS. 

### Root Cause

The for loop from [](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)is missing the increment expression. The for loop becomes an infinite loop and is executed until it runs out of gas. 
All transactions that calls this function, reverts with an out-of-gas error. 

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) { // @audit missing increment
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

`calculateFeeAndNewAmountForOneTVS()` is called from both [mergeTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013) and [splitTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069). Both functions reverts due to infinite loop, blocking the splitting or merging the TVS. 

### Internal Pre-conditions

An user calls `mergeTVS()` or `splitTVS()`. 

### External Pre-conditions

None

### Attack Path

1. An user calls `mergeTVS()` (or `splitTVS()`) to merge 2 of his TVS;
2. `calculateFeeAndNewAmountForOneTVS()` is called to calculate the `feeAmount` and `newAmount`. When this function is executed, it enters the infinite for loop. When all the gas is consumed, tx reverts.

As a results users can't merge or split their TVS. 

### Impact

A core protocol functionality is not working at all. 
Users can't merge or split the TVSs. 

### PoC

_No response_

### Mitigation

Update `calculateFeeAndNewAmountForOneTVS()` function :

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
-        for (uint256 i; i < length;) {
+        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  