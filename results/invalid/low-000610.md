# [000610] Splitting fee can round down to zero in edge case
  
  ### Summary

Due to the fact that there is no scaling in `calculateFeeAmount` function, if a user splits aggressively for any reason, after some point, the protocol will stop receiving fees due to round down to zero.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

### Details

`splitTVS` calls `calculateFeeAndNewAmountForOneTVS` which in turn calculates via `calculateFeeAmount`

The user can decide the allocation percentage for each newly split allocation. This will be calculated in `_computeSplitArrays` function.

**Keep in mind that each split reduces each allocation, as that’s the point. And that there is no limit for splitting.**

Now, regarding the below code block:

```
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

Say that a user, for any reason, did this split multiple times that they came to a point that amount is small enough that the feeAmount calculation returns 0. 

That means that in `calculateFeeAndNewAmountForOneTVS` function, there will be no fee given to the protocol.

### Impact

Likelihood of this happening is low, and the impact is medium. Hence I give low severity for this issue, since there’s loss of revenue for the protocol.

Can provide POC if the judge requires.
  