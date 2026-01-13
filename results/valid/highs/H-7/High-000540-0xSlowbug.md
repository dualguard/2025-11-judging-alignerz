# [000540] Missing Loop Increment Causes Infinite Loop & Guaranteed Revert
  
  ### Summary

The failure to increment the loop counter inside [`calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L175) will cause an infinite loop for all callers, resulting in an inevitable revert as the function consumes all gas.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()` the loop is written as:

```solidity
for (uint256 i; i < length;) {
    ...
}
```

No increment (`i++`) or unchecked increment exists within the loop body.
As a result, `i` remains permanently equal to `0`, and the loop never terminates.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user calls `mergeTVS()` or `splitTVS`.
2. The function loads the TVS amounts and implicitly calls:

   ```solidity
   calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
   ```
3. Execution enters the loop:

   ```solidity
   for (uint256 i; i < length;) {
   ```
4. `i` starts at `0` and never increments.
5. The loop condition is always true.
6. The function continues executing until it runs out of gas.
7. The entire `mergeTVS()` or `splitTVS` call **always** reverts due to OOG.

### Impact

The users cannot merge their TVSs.
All calls to `mergeTVS()` and `splitTVS()` involving any number of vesting flows will always revert due to infinite looping and out-of-gas.

This makes the merge and split functionality completely unusable.


### PoC

_No response_

### Mitigation


Increment the loop counter:

**Either the standard form:**

```solidity
for (uint256 i; i < length; i++) {
    ...
}
```

**Or the gas-optimized version:**

```solidity
for (uint256 i; i < length;) {
    ...
    unchecked { i++; }
}
```
  