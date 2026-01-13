# [000473] The `calculateFeeAndNewAmountForOneTVS`  in `FeesManager` contract doesn't increment the index variable in for loop causing infinite loop which ultimately causes Out-of-gas revert
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function contains a for loop that never increments the loop `counter i`, resulting in a permanent infinite loop.
Because `i` always remains `0`, the condition `i < length` is always true, causing the function to **loop forever until gas is exhausted, which results in a hard revert due to out-of-gas**.

This flaw makes the function completely non-executable, and since both `mergeTVS` and `splitTVS` depend on it, these functionalities also become unusable.

### Root Cause

The loop variable `i` is never incremented inside the for loop, causing an infinite loop:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    for (uint256 i; i < length;) {   //<@ No i++ inside loop body
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
@>  // Missing: i++;
    }
}

```

Link to the affected code:-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169

### Internal Pre-conditions

This doesn't require and pre-conditions.

### External Pre-conditions

This doesn't require and pre-conditions.

### Attack Path

1. User calls `mergeTVS` or `splitTVS` function.
2. `mergeTVS` or `splitTVS` calls `calculateFeeAndNewAmountForOneTVS`  function internally to calculate the fee (either for merge or split).
3. The `calculateFeeAndNewAmountForOneTVS` function creates an infinite loop becuase the counter variable is not incremented, making `i < length` always true. The loop runs again and again until the transaction reverts with Out-of-gas error.

### Impact

1. **DoS (Denial of Service)** – The function never terminates and always hits an out-of-gas revert, making it impossible to compute fees.
2. **Broken merge/split logic** – Both `mergeTVS` and `splitTVS` are permanently unusable since they rely on this function.
3. **Protocol-level disruption** – Any vesting operation needing fee adjustments cannot progress.
4. **Gas Drain** – Any external caller attempting to use this function will waste all gas provided to the call.

**Severity - HIGH**

### PoC

N/A

### Mitigation

To mitigate this issue, just increment the counter variable `i`:-
```diff
function calculateFeeAndNewAmountForOneTVS(...) {
     for (uint256 i; i < length;) {
         feeAmount += calculateFeeAmount(feeRate, amounts[i]);
         newAmounts[i] = amounts[i] - feeAmount;
+        i++; 
     }
}

```
  