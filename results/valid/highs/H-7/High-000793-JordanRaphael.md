# [000793] Missing Loop Increment Causes Permanent Reverts in Fee Calculation
  
  ### Summary

A missing increment in the fee-calculation loop will cause an infinite loop, leading to a complete loss of functionality for users as any caller will trigger a permanent revert when invoking logic that relies on `calculateFeeAndNewAmountForOneTVS`.


### Root Cause

In `FeesManager.sol` the `for` loop inside `calculateFeeAndNewAmountForOneTVS` does not increment the loop counter `i`, causing the loop to run indefinitely and revert due to exceeding the block gas limit.

[Missing index increment](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L170)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Any role (user, protocol component, or external contract) calls a function that internally invokes `calculateFeeAndNewAmountForOneTVS`.
2. Execution reaches the `for` loop inside the function.
3. Since `i` is never incremented, the loop condition `i < length` always holds, causing an infinite loop.
4. The transaction eventually runs out of gas and reverts.
5. All higher-level operations that depend on fee calculation—such as `splitTVS` and `mergeTVS`—become unusable.


### Impact

The protocol and all users cannot execute any functionality that depends on `splitTVS` or `mergeTVS`, effectively disabling critical parts of the system. No PoC is needed, as the issue is a guaranteed revert path. The affected parties lose the ability to perform required operations, blocking core protocol behavior.


### PoC

_No response_

### Mitigation

Increment the loop counter to ensure proper iteration.
Additionally, consider adding tests that cover loop execution paths to prevent similar regressions in the future. 
  