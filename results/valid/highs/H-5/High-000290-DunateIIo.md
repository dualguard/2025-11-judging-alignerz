# [000290] Incorrect Fee Deduction In calculateFeeAndNewAmountForOneTVS() Causing Direct Loss of Tokens When Using mergeTVS() or splitTVS()
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` is the function that calculates the splitting and merging fees, it's mainly a for loop that iterates over the `allocation.amounts` to apply the `feeRate` (both passed as inputs), however, instead of just calculating the fee for each amount, it accumulates the deducted fee every cycle and deducts that accumulated value from every amount.

### Root Cause

The function iterates on this block causing amplified fees every cycle:
https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

After using calling the function by `splitTVS()` or `mergeTVS()`, the outcome is either a fund loss if the newAmount was big enough to survive those exponential fees or to just revert.

### PoC

_No response_

### Mitigation

This is a complete correct function (the other bugs are reported in separate reports)

```
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 totalFeeAmount, uint256[] memory newAmounts) {
    // Fix 1: Initialize the array
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
        // FIX 2: Subtract only the current fee, not accumulative.
        newAmounts[i] = amounts[i] - currentFee;
        totalFeeAmount += currentFee;

        // FIX 3: Increment the counter
        unchecked { ++i; }
    }
}
```
  