# [000938] Uninitialized array in `calculateFeeAndNewAmountForOneTVS` causes out-of-bounds access and DoS
  
  ### Summary

The uninitialized `newAmounts` array in `FeesManager::calculateFeeAndNewAmountForOneTVS` will cause an out-of-bounds memory access for users as any user attempting to split or merge their TVS will trigger an immediate revert when the function attempts to write to the unallocated array.

### Root Cause

In [FeesManager.sol:169](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L173) the newAmounts return variable is declared but never allocated memory. The function attempts to write to `newAmounts[i]` at line 172 without first initializing the array with `newAmounts = new uint256[](length)`
Scenerio: 
- User calls `splitTVS(`) with valid parameters 
- Function calls `calculateFeeAndNewAmountForOneTVS()`  [ AlignerzVesting.sol:1069](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069)
- Loop attempts to write to newAmounts[i] at unallocated memory [FeesManager.sol:172](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172)
- Transaction reverts with out-of-bounds array access



### Impact

Users cannot split or merge their TVS tokens

### PoC

.

### Mitigation

Initialize the newAmounts array before the loop
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {  
    newAmounts = new uint256[](length); // Add this line  
    for (uint256 i; i < length;) {  
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  
        newAmounts[i] = amounts[i] - feeAmount;  
        unchecked { ++i; } // Also fix missing increment  
    }  
}
```
  