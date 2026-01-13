# [000792] Unallocated Array in Fee Calculation Causes Reverts in splitTVS and mergeTVS
  
  ### Summary

An unallocated return array (`newAmounts`) will cause a complete loss of functionality for the protocol as any caller will consistently trigger a revert when executing `splitTVS` or `mergeTVS`, since these functions rely on `calculateFeeAndNewAmountForOneTVS`, which writes to an uninitialized memory array.


### Root Cause

In `FeesManager.sol` the `newAmounts` array is never allocated before being written to inside the loop, causing a panic due to out-of-bounds writes.

[newAmounts unallocated](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L172)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Any role (user, protocol component, or internal caller) invokes `splitTVS` or `mergeTVS`.
2. These functions internally call `calculateFeeAndNewAmountForOneTVS`.
3. Execution reaches the line `newAmounts[i] = amounts[i] - feeAmount;`.
4. Because `newAmounts` has not been allocated with `new uint256[](length)`, writing to index `i` triggers a panic (`0x32: array out-of-bounds access`).
5. The entire transaction reverts, making the functionality unusable.


### Impact

The protocol and all users cannot execute `splitTVS` or `mergeTVS`, blocking critical core operations. No PoC is required, as the operation always reverts due to a guaranteed out-of-bounds write. The affected parties are unable to perform essential actions required for the system to function.


### PoC

_No response_

### Mitigation

Allocate the return array before writing to it.
Adding tests for memory allocation and return-array correctness will help prevent similar regressions in the future.
  