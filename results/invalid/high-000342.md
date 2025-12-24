# [000342] In `AlignerzVesting.sol::mergeTVS`, users will pay 0.5% merging fee for the destination TVS when protocol specification mandates merging is FREE
  
  ### Summary

Incorrect fee application logic in `mergeTVS()` will cause financial loss for users as the protocol applies a 0.5% fee to the destination TVS, when the AlignerZ specification explicitly states that merging should be free.


### Root Cause

In [`AlignerzVesting.sol::mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013), the function incorrectly applies fees twice:

```solidity
    function mergeTVS(...) external returns(uint256) {
        // ...
    
        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
    
        // @audit-issue: Fee applied to destination TVS
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;  // Reduced by 0.5%

        token.safeTransfer(treasury, feeAmount);  // Total fee transferred
         // ...
    }
```

### Internal Pre-conditions

1. User must own multiple TVS NFTs (destination and source TVSs)
2. All TVSs must be from the same project with matching token types
3. User must call `mergeTVS` to combine their TVSs
4. Contract must have sufficient token balance to transfer fees to treasury

### External Pre-conditions

_No response_

### Attack Path

1. User owns TVS A with 1,000 tokens and TVS B with 2,000 tokens (total: 3,000 tokens)
2. User calls `mergeTVS(...)` to merge TVS B into TVS A
3. The function applies 0.5% fee to TVS A: 1,000 × 0.5% = 5 tokens deducted
4. TVS A now has 995 tokens after destination fee

### Impact

Users suffer direct financial loss of 0.5% of their total TVS value due to fee calculations in the `calculateFeeAndNewAmountForOneTVS` helper function.
The AlignerZ protocol specification explicitly states that merging fee is free

### PoC

_No response_

### Mitigation

Remove the fee calculations from the `mergeTVS` function to align with the AlignerZ specification:

```diff
    function mergeTVS(...) external returns(uint256) {
        // ...
    
-        uint256[] memory amounts = mergedTVS.amounts;
-       uint256 nbOfFlows = mergedTVS.amounts.length;
    
        // Remove fee calculation on destination TVS
-        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
-       mergedTVS.amounts = newAmounts; 
         
         // Remove fee transfer to treasury
-        token.safeTransfer(treasury, feeAmount);
         // ...
    }
```
  