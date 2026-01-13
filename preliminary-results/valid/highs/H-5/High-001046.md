# [001046] calculateFeeAndNewAmountForOneTVS causes permanent loss of funds
  
  ### Summary

Current `calculateFeeAndNewAmountForOneTVS` is wrongly accumulating newAmounts for mergedTVS resulting in permanent loss. feeAmount is accumulated while newAmounts[i] is computed by subtracting that cumulative fee from each flow, which charges later flows for earlier flows fees.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Root Cause

`FeeManager::calculateFeeAndNewAmountForOneTVS` computes per-flow fee perFee_i but then does feeAmount += perFee_i and sets newAmounts[i] = amounts[i] - feeAmount (cumulative). 
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
As a result each later flow is reduced by the sum of all previous per-fees plus its own, not by its own per-fee alone.
The contract transfers only feeAmount (sum of perFee_i) to treasury. The extra amount removed from flows (the difference between cumulative-subtractions and the actual fees transferred) is neither recorded nor transferred - users entitlements are silently reduced.
There is underflow risk as well, where `feeAmount` could be > `amounts[i]`, which will cause `mergeTVS` to fail

### Internal Pre-conditions

1. merging on TVS should be called

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Permanent loss of user entitlement: tokens disappear from users claimable balances even though treasury only receives the true fees.
Underflow/revert: if cumulative feeAmount > amounts[i] for some i, the subtraction reverts, causing mergeTVS to fail.

### PoC

Example A - silent loss of value (no revert)

Inputs:
amounts = [100, 100]
feeRate = 100 (1% -> perFee for each flow = floor(100 * 100 / 10000) = 1)
Step-by-step:
i = 0:
feeAmount = 0 + 1 = 1
newAmounts[0] = 100 - 1 = 99
i = 1:
feeAmount = 1 + 1 = 2
newAmounts[1] = 100 - 2 = 98
Results:
fee transferred to treasury (feeAmount) = 2
sum(before) = 200
sum(after) = 99 + 98 = 197
sum(after) + feeTransferred = 197 + 2 = 199
Missing = 200 - 199 = 1 token
Explanation: users lose 1 token of entitlement. That token is not transferred to treasury and is removed from bookkeeping - effectively lost from users claimable balances.

Example B - larger n, larger loss (shows scaling)

Inputs:
amounts = [100, 100, 100]
feeRate = 100 (1% -> perFee = 1 each)
Results:
feeAmount final = 3
newAmounts = [99, 98, 97] (because second subtracts cumulative 2, third subtracts cumulative 3)
sum(after) = 294
sum(after) + feeTransferred = 294 + 3 = 297
Missing = 300 - 297 = 3 tokens lost
Intuition: loss grows with number of flows because each flow is charged earlier flows fees again.

Example C - underflow / revert (when cumulative fee > a later amount)

Inputs:
amounts = [100, 1]
feeRate = 200 (2% -> perFee_0 = floor(100 * 200 / 10000) = 2)
Step-by-step:
i = 0:
feeAmount = 2
newAmounts[0] = 100 - 2 = 98
i = 1:
feeAmount = 2 + perFee_1 (perFee_1 = floor(1 * 200 / 10000) = 0) -> feeAmount stays 2
newAmounts[1] = 1 - 2 -> underflow -> transaction reverts

### Mitigation

Implement per-flow fee subtraction:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        require(length <= amounts.length, "Invalid length");
        newAmounts = new uint256[](length);
        for (uint256 i = 0; i < length;) {
            uint256 perFee = calculateFeeAmount(feeRate, amounts[i]);
            // Subtract per-flow fee, avoid underflow by clamping to zero
            if (amounts[i] >= perFee) {
                newAmounts[i] = amounts[i] - perFee;
            } else {
                newAmounts[i] = 0;
            }
            feeAmount += perFee;
            unchecked {
                ++i;
            }
        }
        return (feeAmount, newAmounts);
    }
```
  