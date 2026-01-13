# [000351] [M-2] Uninitialized memory array in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS#169` will cause memory out-of-bounds and revert
  
  ### Summary

Failure to initialize the `newAmounts` memory array in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS#169`  will cause an immediate revert because in Solidity, we can't write to an unallocated memory array

### Root Cause

The [`FeesManager.sol::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169), function tries to write to an unallocated memory array, which causes a revert

```solidity
    function calculateFeeAndNewAmountForOneTVSA(...) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length; i++) {
            // ...
            // @audit-issue: writing to an unllocated memory array. This will always revert
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Internal Pre-conditions

1. At first the function calls the `for` loop, which executes normally, until it tries to write to the `newAmounts` memory, which causes a revert

### External Pre-conditions

_No response_

### Attack Path

1. `newAmounts` is declared but never initialized.
2. At first, the code runs well.
3. When Solidity try to write to `newAmounts`.
5. The transaction reverts instantly.

### Impact

This causes a complete halt in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS`, and any dependent logic causing complete disruptions to the protocol.
For example both `AlignerzVesting.sol::mergeTVS#1002` and `AlignerzVesting.sol::splitTVS#1054` calls this.

### PoC

_No response_

### Mitigation

```solidity    
function calculateFeeAndNewAmountForOneTVS(...) public pure returns (...) {
        // Add this line to initialize the memory array.
        newAmounts = new uint256[](length);
        for (uint256 i; i < length; ) {
            // ...
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  