# [000087] Uninitialized array causes function to always revert
  
  ### Summary

The failure to initialize the `newAmounts` array in `calculateFeeAndNewAmountForOneTVS` will cause all merge and split operations to revert as users attempt these operations, since the function tries to write to an unallocated memory array.

### Root Cause

In [FeesManager.sol:169-174](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the `calculateFeeAndNewAmountForOneTVS` function declares` newAmounts` as a return parameter but never allocates memory for it.  When the function attempts to write `newAmounts[i] = amounts[i] - feeAmount` , it tries to write to an uninitialized array, causing an out-of-bounds revert. 

### Internal Pre-conditions

1. User needs to call `mergeTVS()` or `splitTVS()` which internally call `calculateFeeAndNewAmountForOneTVS()`

### External Pre-conditions

None 

### Attack Path

1. User calls `mergeTVS()` with valid NFT IDs to merge allocations
2. Function calls `calculateFeeAndNewAmountForOneTVS()` at line 1013
3. Inside the helper function, the loop attempts to write to `newAmounts[i]` 
4. Transaction reverts due to writing to unallocated memory
5. User cannot merge or split any NFTs

### Impact

Complete DoS of merge and split functionality. Users cannot:

- Merge multiple vesting NFTs into one
- Split a vesting NFT into multiple parts
- Perform any portfolio management operations

This renders critical protocol features completely unusable.

### PoC

The bug is self-evident from code inspection.

### Mitigation

Initialize the newAmounts array before the loop:

```diff
function calculateFeeAndNewAmountForOneTVS(  
    uint256 feeRate,  
    uint256[] memory amounts,  
    uint256 length  
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {  
+   newAmounts = new uint256[](length); // Add this line  
    for (uint256 i; i < length; ) {  
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  
        newAmounts[i] = amounts[i] - feeAmount;  
        unchecked { ++i; } // Also fix the infinite loop  
    }  
}
```
  