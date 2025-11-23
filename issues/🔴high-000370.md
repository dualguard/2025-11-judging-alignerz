# [000370] H08 Unallocated memory in FeesManager causes DoS in splitTVS and mergeTVS
  
  ## Summary

The `FeesManager` contract contains a critical bug in `calculateFeeAndNewAmountForOneTVS` where it returns a memory array `newAmounts` without allocating memory for it. This causes `splitTVS` and `mergeTVS` to revert due to invalid memory access, rendering these core features unusable.

## Root Cause

In `FeesManager.sol`:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
1. @>       newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

The return variable `newAmounts` is declared as `uint256[] memory` but is never initialized with `new uint256[](length)`. In Solidity, declaring a memory array does not allocate space for it. Accessing `newAmounts[i]` at marker 1 attempts to write to a null or invalid memory pointer, causing the transaction to panic with an "Array Out of Bounds".

## Internal Pre-Conditions

1.  `splitFeeRate` or `mergeFeeRate` must be non-zero (or even zero, as the loop runs regardless).
2.  The function must be called with `length > 0`.

## External Pre-Conditions

None.

## Attack Path

1.  User calls `splitTVS` or `mergeTVS`.
2.  `calculateFeeAndNewAmountForOneTVS` is called.
3.  The loop attempts to write to `newAmounts[0]`.
4.  The transaction reverts immediately.

## Impact

**High**. The `splitTVS` and `mergeTVS` functions are completely broken. Users cannot split or merge their vesting positions, leading to a Denial of Service of these features.

## PoC

Trivial.

## Mitigation

Allocate memory for `newAmounts` before the loop.

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            // ...
        }
    }
```

  