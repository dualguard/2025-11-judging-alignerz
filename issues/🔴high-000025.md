# [000025] Incorrect cumulative fee deduction corrupts TVS amounts in merge/split operations
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` helper applies fees incorrectly by subtracting the **cumulative** `feeAmount` from each flow’s amount, instead of subtracting only the fee for that specific flow:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // <---- here
    }
}
```

Here:

* `feeAmount` is accumulated across iterations (sum of all previous fees).
* `newAmounts[i]` uses `amounts[i] - feeAmount` instead of `amounts[i] - feeForThisFlow`.
* As a result, later flows are charged not only their own fee but also all previously accrued fees.

Because this function is used in merge/split TVS flows, it directly affects users’ vesting token balances and the fee amounts collected by the protocol, forcing users to pay more fees than intended and corrupting the total balances.

### Root Cause

The function subtracts the cumulative fee from each flow instead of the per-flow fee, causing incorrect amount calculations during merge/split, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Users are overcharged and receive incorrect vesting balances, breaking value conservation and leading to potential loss of funds.

### PoC

_No response_

### Mitigation

Consider updating the function to compute and subtract the **per-flow** fee, not the cumulative fee.
  