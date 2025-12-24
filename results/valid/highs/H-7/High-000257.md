# [000257] Infinite Loop in `Feesmanager::calculateFeeAndNewAmountForOneTVS` Causes DOS in `AlignerzVesting::splitTvs` and `AlignerzVesting::mergeTvs`
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

The function `Feesmanager::calculateFeeAndNewAmountForOneTVS` contains a `for` loop where the loop counter `i` is never incremented. As a result, the loop never progresses, leading to an infinite loop and an out of gas revert.

Because `AlignerzVesting::splitTvs` and `AlignerzVesting::mergeTvs` both depend on this function, any call into them becomes permanently unexecutable, resulting in a denial of service for core protocol operations.

### Root Cause

In the below affected function:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) { // Missing increment: i++
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```
the loop lacks an increment statement:
```solidity
// Missing increment: i++
```
This means:
- `i` remains 0 forever
- `i < length` is always true
- The function never terminates
- Execution runs until the EVM runs out of gas and reverts


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- User or contract calls `AlignerzVesting::splitTvs` or `AlignerzVesting::mergeTvs`
- `AlignerzVesting::splitTvs` / `AlignerzVesting::mergeTvs` forwards inputs to `Feesmanager::calculateFeeAndNewAmountForOneTVS`
- Execution enters the for loop:
    The loop condition is:
    ```solidity
    for (uint256 i; i < length;) {}
    ```
    Since `i` starts at 0 and is never incremented, the condition `i` < length is always true.
- Loop never progresses
- Transaction hits out of gas and reverts
- The EVM aborts execution and reverts the entire transaction.

### Impact

- Users cannot merge Tokenized Vesting Schedules
- Users cannot split Tokenized Vesting Schedules
- Total denial of service to `AlignerzVesting::splitTvs` and `AlignerzVesting::mergeTvs`
- Treasury fee logic cannot run

### PoC

_No response_

### Mitigation

Add the missing increment:
```solidity
for (uint256 i; i < length; i++) {}
```
  