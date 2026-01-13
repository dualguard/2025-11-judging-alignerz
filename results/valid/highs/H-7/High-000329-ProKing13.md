# [000329] Infinite Loop causes Permenant DoS in Fee Calculation Function
  
  ### Summary

The [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function contains an infinite loop that causes all merge and split operations to fail with out-of-gas errors. The loop counter `i` is never incremented, causing the loop to run indefinitely until gas is exhausted. This makes the `mergeTVS` and `splitTVS` functions completely unusable, resulting in a permanent denial of service for core protocol features.

### Root Cause

The `for` loop in `calculateFeeAndNewAmountForOneTVS` is missing the increment statement for the loop counter:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        ...
        ...
   --->     // Missing: i++ or ++i
    }
}
```

The loop initializes `i = 0` and checks `i < length`, but never increments `i`. This causes the condition to remain true forever, creating an infinite loop that runs until the transaction runs out of gas.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


This is not an attack but a bug affecting all users:

1. User calls `mergeTVS(projectId, mergedNftId, projectIds, nftIds)` or `splitTVS(projectId, percentages, splitNftId)`
2. Function reaches the call to `calculateFeeAndNewAmountForOneTVS(feeRate, amounts, length)`
3. Loop starts with `i = 0`
4. Loop condition `0 < length` is true
5. Loop body executes (calculates fee)
6. Loop counter `i` remains 0 (no increment)
7. Loop condition `0 < length` is still true
8. Loop repeats indefinitely
9. After iterations, transaction runs out of gas
10. Transaction reverts with out-of-gas error
11. User loses gas fees
12. Merge/split operation fails completely

### Impact


- Permanent DoS
- `mergeTVS` function is completely unusable (100% failure rate)
- `splitTVS` function is completely unusable (100% failure rate)
- Users lose gas fees per failed transaction



### PoC

_No response_

### Mitigation

Add the missing loop counter increment

  