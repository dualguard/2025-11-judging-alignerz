# [000255] `FeesManager::calculateFeeAndNewAmountForOneTVS` Computes Incorrect Values Due to Cumulative Fee Deduction
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L171-L172

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013

The function `calculateFeeAndNewAmountForOneTVS` is intended to apply a fee *per flow* and return a new adjusted `amounts` array, where each `newAmounts[i] = amounts[i] - fee(amounts[i])`.  
However, the implementation subtracts the **running cumulative fee** instead of the individual fee for that index:

```solidity
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount; // incorrect
```
This causes incorrect outputs for all flows after index 0.  
This logic error propagates into `mergeTVS` and `splitTVS`, causing unexpected reductions in vesting amounts, violating user expectations and leading to silent value loss.

### Root Cause

The Fee Is Incorrectly Accumulated Across Iterations.

Inside the loop:
```sol
feeAmount += calculateFeeAmount(feeRate, amounts[i]);
newAmounts[i] = amounts[i] - feeAmount;
```
The fee for flow `i` should be:
```sol
fee_i = calculateFeeAmount(feeRate, amounts[i])
newAmounts[i] = amounts[i] - fee_i

```

Instead, the implementation subtracts:
```js
(amounts[i] - sum(fees[0..i]))

```

This produces incorrect results and breaks deterministic vesting arithmetic.

## Failure Path

### In `mergeTVS` and `splitTvs`

They both calls:
```js
(uint256 feeAmount, uint256[] memory newAmounts) =
    calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);

(uint256 feeAmount, uint256[] memory newAmounts) =
    calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);

```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- A user performs either `mergeTVS` or `splitTVS`, both of which call `calculateFeeAndNewAmountForOneTVS`.
- For the first flow, the fee is correct.
- For every subsequent flow, the function subtracts the cumulative fee instead of the per flow fee.
- This causes each later flow’s amount to be reduced by more than intended, producing incorrect `newAmounts`.
- The protocol accepts these incorrect values without validation, resulting in silent and permanent loss of vesting amounts for users interacting with merge/split operations.

### Impact

Incorrect cumulative fee deduction silently reduces user balances across all flows after index 0. This leads to unintended value loss in both `mergeTVS` and `splitTVS`, breaking expected vesting math and causing users to receive less than they should.

### PoC


```solidity
function test_FeesManager_calculateFeeAndNewAmountForOneTVS_behavior() public {
    uint256;
    amounts[0] = 100 ether;
    amounts[1] = 200 ether;

    uint256 feeRate = 1000; // 10%

    (uint256 totalFee, uint256[] memory newAmounts) =
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 2);

    // Expected:
    // fee on 100 = 10
    // fee on 200 = 20
    // total = 30
    assertEq(totalFee, 30 ether);
    assertEq(newAmounts[0], 90 ether);
    assertEq(newAmounts[1], 180 ether);
}

```

### Mitigation

Apply fees per flow (not cumulatively)
Correct implementation:

```sol
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);

    for (uint256 i; i < length; i++) {
        uint256 feeForFlow = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += feeForFlow;
        newAmounts[i] = amounts[i] - feeForFlow;
    }
}

```
  