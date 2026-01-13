# [000533] calculateFeeAndNewAmountForOneTVS incorrect implementation
  
  ## Summary

Due to incorrect implementation of the function `calculateFeeAndNewAmountForOneTVS()` the functions `mergeTVS()` and `splitTVS()` cannot be executed.



## Root cause

The function `calculateFeeAndNewAmountForOneTVS()` always be reverted.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

First, the function has misses incrementing `i`. The for-loop is infinite and consequently always reverts.

Second, `newAmounts` is declared in the return type but not initialized inside the function. Memory arrays must be explicitly allocated with a size (e.g., `new uint256[](size)`).

Next, here is incorrect `fee` counting logic. This subtracts the cumulative `feeAmount` (total fees so far, including previous iterations) from each `amounts[i]`. But it must subtract only the per-item fee, not the cumulative total.



## Impact

The functions `mergeTVS()` and `splitTVS()` are always reverted and cannot be executed. Incorrect fee calculations will lead to the loss of funds.



## Mitigation

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;
            unchecked {
                ++i;
            }
        }
    }
```

  