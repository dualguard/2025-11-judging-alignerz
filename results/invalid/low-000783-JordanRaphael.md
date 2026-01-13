# [000783] Unbounded Event Emissions via Zero-Percentage Splits in splitTVS
  
  ### Summary

A lack of input validation will cause excessive event pollution for the protocol as the NFT owner will call `splitTVS` with zero-percentage allocations, emitting countless TVSSplit events.

### Root Cause

In `AlignerzVesting.sol` the `splitTVS` function lacks validation to prevent zero-percentage splits. The function processes all percentages in the input array, including zeros, and emits a `TVSSplit` event for each one. Since the only validation is that percentages sum to `BASIS_POINT`, an attacker can pass arrays containing many zero-percentage entries that will be processed normally.

[splitTVS](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107)


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. NFT owner calls `splitTVS` with a `percentages` array containing many entries with value 0
2. For each percentage (including zeros), the function creates a new NFT and calls `_computeSplitArrays` to allocate zero amounts across the flows
3. A `TVSSplit` event is emitted for each zero-percentage split, even though no meaningful allocation occurred
4. By repeating this with large arrays, the attacker floods the event log with unnecessary TVSSplit and TVSsMerged events


### Impact

The protocol suffers from event log pollution. Event indexing systems and off-chain monitoring tools may experience degraded performance when processing excessive events. There is no direct financial loss, but operational efficiency is degraded.


### PoC

_No response_

### Mitigation

Add a validation check to skip or reject zero-percentage entries in the `splitTVS` function.
Filter out zero percentages before processing, or revert if a percentage is equal to 0.
  