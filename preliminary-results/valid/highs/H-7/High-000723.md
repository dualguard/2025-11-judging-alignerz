# [000723] Missing loop increment in `calculateFeeAndNewAmountForOneTVS` causes infinite loop
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` contains a for loop that never increments its counter variable, causing an infinite loop. This results in all transactions calling `splitTVS` or `mergeTVS` to run out of gas and revert, completely breaking both split and merge functionalities for all users.

### Root Cause

In the `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol`, the for loop declares a counter variable `i` but never increments it within the loop body. This causes the loop condition `i < length` to never become false, resulting in an infinite loop that consumes all available gas.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {  // i starts at 0
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // Missing: i++ or unchecked { ++i; }
    }  // Loop repeats with i still = 0, forever
}
```

This function is called by both `splitTVS` and `mergeTVS` in `AlignerzVesting.sol`, causing both functions to be completely unusable.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User owns a vesting NFT with 1 flow containing 1000 tokens
2. User calls `splitTVS(projectId, [5000, 5000], nftId)` with sufficient gas (e.g., 3,000,000 gas)
3. Function calls `calculateFeeAndNewAmountForOneTVS(200, [1000], 1)`
4. Loop begins with `i = 0, length = 1`:
   - Iteration 1: Processes `amounts[0]`, but `i` is not incremented
   - Iteration 2: `i` still equals 0, processes `amounts[0]` again
   - Iteration 3: `i` still equals 0, processes `amounts[0]` again
   - This continues indefinitely
5. After ~3,000,000 gas is consumed, transaction reverts with "out of gas"
6. User loses all gas fees paid

### Impact

 `splitTVS` and `mergeTVS` are unusable.

### PoC

_No response_

### Mitigation

Add the missing loop increment:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
+       unchecked {
+           ++i;
+       }
    }
}
```
  