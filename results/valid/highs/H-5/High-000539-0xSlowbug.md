# [000539] Cumulative Fee Miscalculation Across Flows
  
  ### Summary

The incorrect use of cumulative fee accumulation inside `calculateFeeAndNewAmountForOneTVS()` will cause inaccurate vesting reductions for users as the fee logic will subtract prior flow fees from subsequent vesting flows during the merge or split process.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()` the fee for a flow is calculated correctly, but then the *cumulative* fee is subtracted from each `amounts[i]`:

```solidity
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;  // ❌ subtracts all previous fees too
```

Correct logic should subtract *only* the fee for the current index, not the accumulated total.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A user calls `mergeTVS()` or `splitTVS()` with an NFT that contains multiple vesting flows.
2. The function loads `mergedTVS.amounts` and calls:

   ```solidity
   calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)
   ```
3. On the first iteration:

   * fee = X
   * newAmount[0] = amounts[0] – X
4. On the second iteration:

   * fee = X
   * cumulative fee = 2X
   * newAmount[1] = amounts[1] – **2X** (incorrect)
5. On the third iteration:

   * cumulative fee = 3X
   * newAmount[2] = amounts[2] – **3X**
6. Vesting flows become progressively more incorrect with each iteration.
7. `_merge()` or `_split()` is then executed using incorrect amounts, permanently corrupting vesting data. Also can cause reverts if the Next amount is less than the cummulative fee been deducted

### Impact

The users suffer an incorrect reduction of vesting amounts, where every flow after the first is overcharged by all previous flows’ fees.
The effective loss grows linearly with the number of vesting flows.

### PoC

_No response_

### Mitigation

Use per-element fee subtraction:

```solidity
uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
feeAmount += fee;
newAmounts[i] = amounts[i] - fee;
```
  