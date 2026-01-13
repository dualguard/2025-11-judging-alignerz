# [000053]  Incorrect Fee Calculation in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;       // @<<<< Uses cumulative feeAmount
    }
}
```

 The function accumulates `feeAmount` across all iterations, but uses the cumulative value to subtract from each individual amount. This means:
- First amount: `newAmounts[0] = amounts[0] - fee0` ✅ Correct
- Second amount: `newAmounts[1] = amounts[1] - (fee0 + fee1)` ❌ Wrong!


### Root Cause



```solidity
 
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;   // @<<<< Uses cumulative feeAmount
    }
}

function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
    feeAmount = amount * feeRate / BASIS_POINT;
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172



### Impact

Incorrect calculation

### PoC

Suppose:

```text
feeRate = 10% (1000 bps)
amounts = [10, 20, 30]
BASIS_POINT = 10_000
```

Step-by-step

* i = 0:

  * `itemFee = 10 * 10% = 1`
  * `feeAmount = 0 + 1 = 1`
  * `newAmounts[0] = 10 - 1 = 9` :white_check_mark: correct

* i = 1:

  * `itemFee = 20 * 10% = 2`
  * `feeAmount = 1 + 2 = 3` (cumulative!)
  * `newAmounts[1] = 20 - 3 = 17` :x: should be `20 - 2 = 18`

* i = 2:

  * `itemFee = 30 * 10% = 3`
  * `feeAmount = 3 + 3 = 6`
  * `newAmounts[2] = 30 - 6 = 24` :x: should be `30 - 3 = 27`

**Result from buggy code:**

```
newAmounts = [9, 17, 24]
```

**Correct result **

```
newAmounts = [9, 18, 27]
```


### Mitigation

Calculate and subtract per item fee inside the loop instead of cumulative fee
  