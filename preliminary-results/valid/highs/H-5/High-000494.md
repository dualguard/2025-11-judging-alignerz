# [000494] Method FeesManager::calculateFeeAndNewAmountForOneTVS() applies fee incorrectly
  
  ### Summary

The below method `FeesManager::calculateFeeAndNewAmountForOneTVS()` subtracts cumulative `feeAmount` on each iteration. Instead of subtracting just `calculateFeeAmount(feeRate, amounts[i]);`.

So, if let's say feeRate is 10% and amounts = [100,200]
```log
First Iteration:
feeAmount = 10% of 100 = 10
newAmounts[0] = 100 - 10 = 90

Second Iteration:
feeAmount = 10 + (10% 0f 200) = 30
// Below instead of subtracting 20 it subtracted 30 which if cumulative fee
newAmounts[1] = 200 - 30 = 170
```
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
@>            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Root Cause

The feeAmount is the sum of fees, While with `newAmounts[i] = amounts[i] - feeAmount;` we are subtracting it from each amount.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

None, It'll happen each time `calculateFeeAndNewAmountForOneTVS()` is called with `length > 1`

### Impact

There can be 2 impacts:

1. The final amounts assigned to each NFT after `mergeTVS()` and `splitTVS()` will be less than they should be.
2. The transaction will revert because **feeAmount > amounts[i]** in `newAmounts[i] = amounts[i] - feeAmount;`.

### PoC

_No response_

### Mitigation

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
++           uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
++           feeAmount += fee;
++           newAmounts[i] = amounts[i] - fee;
--           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
--           newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  