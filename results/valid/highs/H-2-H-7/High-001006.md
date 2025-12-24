# [001006] Uninitialized Array and Missing Loop Increment in Fee Calculation Breaks core operations
  
  ### Summary

The missing loop increment and uninitialized return array in [FeesManager.sol::calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170) will cause an immediate revert or infinite loop consuming all gas for any user attempting to merge or split their TVS NFTs.

### Root Cause

In the calculate function the loop counter i is never incremented, causing an infinite loop

```solidity 
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // newAmounts never initialized
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // Reverts: array length is 0
        MISSING INCREMENT - infinite loop
    }
}
```

### Internal Pre-conditions

User needs to call mergeTVS() or splitTVS() functions
Any non-zero mergeFeeRate or splitFeeRate set by admin

### External Pre-conditions

None

### Attack Path

1. User calls mergeTVS() to combine multiple vesting positions
2. Function calls calculateFeeAndNewAmountForOneTVS() at line 1062
3. Loop runs but i never increments: i=0 → check i < length (true) → execute → i=0 → check again (true) → infinite loop
4. Transaction consumes all gas (~30M) and reverts with "out of gas"

### Impact

- function dos as user  cannot combine TVS positions and separate vesting flows
- Two advertised core features are completely broken

### PoC

_No response_

### Mitigation

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
>    newAmounts = new uint256[](length);  
    
    for (uint256 i; i < length;) {
        // rest of code 
>        unchecked { ++i; }  
    }
}
```
  