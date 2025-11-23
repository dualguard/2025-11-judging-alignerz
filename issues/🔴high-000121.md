# [000121] Misconfigured fee helper can silently lock user funds during merge/split
  
  ### Summary

calculateFeeAndNewAmountForOneTVS() in FeesManager is currently broken by an uninitialized newAmounts array and a missing loop increment, but even if those mechanical issues are fixed, the core fee logic itself is still incorrect. The function subtracts the cumulative fee from each flow instead of the per‑flow fee, which causes users’ TVS amounts to disappear from all allocations without being credited as fees. This is high severity because it leads to permanent loss of user yields and creates surplus tokens that can be withdrawn by the protocol.

### Root Cause

The helper does:

```solidity
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;
```

so later flows pay all previously accumulated fees again. For two flows with amounts `a = b = 100` and 10% fee:

- Per-flow fees: `f0 = f1 = 10`, total `feeAmount = 20` (correct total fee).  
- New amounts: `new0 = 100 − 10 = 90`, `new1 = 100 − (10 + 10) = 80`.  
- Original total: `200`, but `new0 + new1 + feeAmount = 90 + 80 + 20 = 190`.

The missing 10 tokens remain in the contract, not recorded in any allocation or fee variable. With more flows, the shortfall scales further.



### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

When this helper is used by `mergeTVS()` / `splitTVS()`, users’ vesting flows are consistently under-allocated relative to their original amounts, while the contract’s token balance grows by an unrecorded surplus. This causes a loss of yield for users

### PoC

_No response_

### Mitigation

Correct the fee logic to apply only the per-flow fee to each element:

```solidity
newAmounts = new uint256[](length);
for (uint256 i; i < length; ) {
    uint256 perFlowFee = calculateFeeAmount(feeRate, amounts[i]);
    feeAmount += perFlowFee;
    newAmounts[i] = amounts[i] - perFlowFee;
    unchecked { ++i; }
}
```
  