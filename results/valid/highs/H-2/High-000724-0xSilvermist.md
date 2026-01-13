# [000724] Uninitialized return array in `calculateFeeAndNewAmountForOneTVS` causes DoS
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` declares a return parameter `newAmounts` as a memory array but never initializes it with a proper length. When the function attempts to write to this zero-length array, it causes an out-of-bounds revert, making all split and merge operations completely unusable and breaking two core protocol features.

### Root Cause

In the `calculateFeeAndNewAmountForOneTVS` function, the return parameter `uint256[] memory newAmounts` is declared but never initialized with a length before attempting to write to its elements.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // newAmounts is declared but has length 0!
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // REVERTS: writing to uninitialized array
    }
}
```

In Solidity, when a memory array is declared as a return parameter without explicit initialization, it has a default length of zero. Any attempt to access or write to `newAmounts[i]` when `newAmounts.length == 0` will cause an out-of-bounds access revert.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User owns a vesting NFT from any project
2. User calls `splitTVS` to split their NFT
3. Function enters `calculateFeeAndNewAmountForOneTVS`
4. Inside `calculateFeeAndNewAmountForOneTVS`, the loop tries to execute: `newAmounts[0] = ...`
5. Since `newAmounts.length == 0` (uninitialized), this causes an out-of-bounds revert

### Impact

All calls to `splitTVS` and `mergeTVS` will revert, making it impossible for any user to split their vesting NFTs.



### PoC

_No response_

### Mitigation

Initialize the `newAmounts` array with the correct length before attempting any write operations:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```
  