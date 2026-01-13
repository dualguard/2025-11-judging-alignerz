# [000941] Missing loop increment in `calculateFeeAndNewAmountForOneTVS` causes infinite loop and gas exhaustion
  
  ### Summary

The missing loop increment in `FeesManager::calculateFeeAndNewAmountForOneTVS` will cause infinite loop execution for users as any user attempting to split or merge their TVS will consume all available gas when the loop variable i is never incremented. 

### Root Cause

In  [FeesManager.sol:170-173](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170-L173)  the loop variable i is never incremented. The loop condition `i < length` remains perpetually true, causing the contract to execute indefinitely until all gas is consumed. The missing increment should be `unchecked { ++i; }` at the end of the loop body.
 Scenerio: 
- User calls `mergeTVS()` to combine two TVS tokens 
- Function calls `calculateFeeAndNewAmountForOneTVS()` at line 
- Loop enters with i 
- Loop body executes but i is never incremented
- Loop condition i < length remains true forever

### Impact
- The loop condition i < length remains true, causing the contract to run until it consumes all provided gas.




### Mitigation

Add the missing loop increment
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {  
     
    for (uint256 i; i < length;) {  
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  
        newAmounts[i] = amounts[i] - feeAmount;  
        unchecked { ++i; } // add missing increment  
    }  
}
```
  