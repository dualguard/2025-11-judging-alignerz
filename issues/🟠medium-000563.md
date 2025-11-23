# [000563] Integer Division Rounding Down in Fee Calculation Causes Cumulative Revenue Loss for Protocol
  
  
## Summary

Integer division in `calculateFeeAmount` rounds down fee amounts, causing cumulative revenue loss for the protocol. Each split or merge operation loses the remainder from fee calculations, and these losses accumulate over time.

## Root Cause

In [`FeesManager.sol:177`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L177), integer division rounds down the fee calculation. The formula `feeAmount = amount * feeRate / BASIS_POINT` truncates any remainder, so the protocol collects less than the intended fee percentage.

Additionally, in [`FeesManager.sol:169-173`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L173), `calculateFeeAndNewAmountForOneTVS` accumulates `feeAmount` across iterations but subtracts the accumulated total from each individual amount, which is incorrect.

```js
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }
```

## Internal Pre-conditions

1. `splitFeeRate` or `mergeFeeRate` must be set to a non-zero value
2. Users must call `splitTVS()` or `mergeTVS()` with amounts where `amount * feeRate` is not evenly divisible by `BASIS_POINT` (10,000)
3. The protocol must process multiple split or merge operations over time

## External Pre-conditions

None

## Attack Path

1. A user calls `splitTVS()` or `mergeTVS()` with token amounts
2. The protocol calculates fees using `calculateFeeAmount()`, which performs integer division
3. When `amount * feeRate` is not divisible by `BASIS_POINT`, the remainder is lost
4. The protocol transfers the rounded-down fee amount to the treasury
5. This process repeats for each split/merge operation, accumulating lost revenue

## Impact

The protocol suffers cumulative revenue loss. Each transaction loses up to `(BASIS_POINT - 1)` wei per calculation. For example, with `amount = 100`, `feeRate = 1` (0.01%), the expected fee is `0.01`, but integer division yields `0`, losing the entire fee. With larger amounts and higher fee rates, losses scale proportionally. Over many operations, these losses accumulate, reducing protocol revenue.

## Proof of Concept

N/A

## Mitigation

Use rounding up for fee calculations to ensure the protocol collects at least the intended fee amount. Implement a function that rounds up:
  