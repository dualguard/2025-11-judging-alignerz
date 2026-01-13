# [000086] Infinite loop causes transaction to run out of gas
  
  ### Summary

The missing loop increment in `calculateFeeAndNewAmountForOneTVS()` will cause all merge and split operations to run out of gas as users attempt these operations, since the loop variable **i** is never incremented, creating an infinite loop

### Root Cause

In F[eesManager.sol:170-173](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170-L173), the `calculateFeeAndNewAmountForOneTVS()`  has a for loop that never increments the loop variable i.  The loop condition `i < length` will always be true (assuming length > 0), causing the loop to execute indefinitely until the transaction runs out of gas.

### Internal Pre-conditions

1. User needs to call `mergeTVS()` or `splitTVS()` with at least one flow in the allocation
2. The `length` parameter must be greater than 0

### External Pre-conditions

None

### Attack Path

1. User calls `mergeTVS()` with valid NFT IDs containing allocations
2. Function calls `calculateFeeAndNewAmountForOneTVS()` with length > 0 
3. Loop starts with `i = 0` and condition `i < length` evaluates to true
4. Loop body executes but `i` is never incremented
5. Loop repeats infinitely with `i` always equal to 0
6. Transaction runs out of gas and reverts

### Impact

Complete DoS of merge and split functionality. 
Users cannot perform any portfolio management operation:
- merge will revert out of gas
- split will revert out of gas

### PoC

The bug is self-evident from code inspection.

### Mitigation

Add loop increment inside an unchecked block:

```diff
function calculateFeeAndNewAmountForOneTVS(  
    uint256 feeRate,  
    uint256[] memory amounts,  
    uint256 length  
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {  
    for (uint256 i; i < length; ) {  
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  
        newAmounts[i] = amounts[i] - feeAmount;  
+       unchecked { ++i; } // Add this line  
    }  
}

```
  