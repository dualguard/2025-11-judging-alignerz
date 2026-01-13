# [000831] Incorrect `newAmount` calculation in `calculateFeeAndNewAmountForOneTVS()`, causing wrong total fee
  
  
## Summary

Subtracting accumulated fees instead of individual fees in `calculateFeeAndNewAmountForOneTVS()` will cause users to lose significantly more tokens than intended when splitting or merging their TVS.

## Root Cause

In [FeesManager.sol:169-174](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the function accumulates fees in `feeAmount` but subtracts this accumulated total from each individual amount:

```solidity
function calculateFeeAndNewAmountForOneTVS(...) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

The issue is that `newAmounts[i]` subtracts `feeAmount` which contains the sum of fees from all previous iterations (0 to i), when it should only subtract the fee for the current amount.

## Internal Pre-conditions

1. User needs to call `mergeTVS()` or `splitTVS()` with a TVS that has at least 2 flows
2. Split fee or merge fee needs to be set to any value greater than 0

## External Pre-conditions

None

## Attack Path

This is not an attack but a vulnerability in the fee calculation logic that affects all users:

1. User calls `splitTVS()` with their NFT that has 2 vesting flows of 1000 tokens each
2. The protocol applies a 1% split fee
3. For flow 0: fee = 10, accumulated = 10, `newAmount[0] = 1000 - 10 = 990` (correct)
4. For flow 1: fee = 10, accumulated = 20, `newAmount[1] = 1000 - 20 = 980` (should be 990)

The same issue occurs in `mergeTVS()`.

## Impact

Users suffer significant loss that grows with the number of flows when splitting or merging their TVS.

## Mitigation

Use temp variable `currentFee` for `newAmounts` calculation:

```diff
function calculateFeeAndNewAmountForOneTVS (...) {
    for (uint256 i; i < length;) {
-       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-       newAmounts[i] = amounts[i] - feeAmount;
+       uint256 currentFee = calculateFeeAmount(feeRate, amounts[i])
+       feeAmount += currentFee;
+       newAmounts[i] = amounts[i] - currentFee;
+       unchecked { ++i; }
    }
}
```

  