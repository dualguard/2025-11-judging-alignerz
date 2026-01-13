# [000185] `calculateFeeAndNewAmountForOneTVS` function in FeesManager contract incorrectly substract `feeAmount`, leading to loss of user unclaimed funds.
  
  ### Summary

The  `calculateFeeAndNewAmountForOneTVS` function, defined in _FeesManager_ contract, is called by both `mergeTVS` and `splitTVS` functions and is defined as follows:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

Multiple findings are related to this function (`newAmounts` memory array not initialized leading to out of bounds panic,  missing `i` incrementation), but the problem with the current finding is that the logic with `feeAmount`  is incorrect.

### Root Cause

The root cause is that for each iteration of loop, the code does:

```solidity
            newAmounts[i] = amounts[i] - feeAmount;
```

But `feeAmount` is incremented during each iteration of the loop. This is wrong, and means that the bigger `amounts` array is, the bigger the loss is.

Indeed, the first flow of the TVS will be correctly computed. But the amount of the second flow will be reduced by the fee corresponding to the current amount + the fee of the previous flow,  and so on.


### Attack Path

There is no specific attack path. There is a bug in the way fees are subtracted from TVS amounts.

### Impact

The impact of this issue is high as it leads to loss of rewards/funds for users. This is not really rewards in the case of bidding projects because users pay stablecoins to the protocol to get their allocation.

### Mitigation

Make sure to correctly subtract fees to the `amounts` elements:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length); // FOR TEST

        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;

            // FOR TEST
            unchecked {
                ++i;
            }
        }
    }
```
  