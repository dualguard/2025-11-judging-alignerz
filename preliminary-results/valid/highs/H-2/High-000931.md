# [000931] Uninitialized Memory Array in calculateFeeAndNewAmountForOneTVS() Causes Immediate Revert
  
  ### Summary


The function [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-#L173) declares newAmounts without allocating memory. Any attempt to write to this array immediately causes an out-of-bounds memory write and triggers a full revert. Because mergeTVS() and splitTVS() call this function at the start of their execution, both features always revert and are completely unusable.


### Root Cause

In FeesManager.sol:

```
uint256[] memory newAmounts; // uninitialized
...
newAmounts[i] = amounts[i] - feeAmount; // <-- immediate revert
```


The array is never allocated:

`newAmounts = new uint256[](length);`


Not calling this causes:

* First write to newAmounts[i] → memory OOB → panic

* Transaction immediately reverts

This happens before any other logic executes.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Failure Path

1. User calls merge/split.

2. Vesting contract calls calculateFeeAndNewAmountForOneTVS().

3. First write to newAmounts[i] attempts writing into uninitialized memory.

4. EVM throws memory OOB panic (0x31).

5. Entire transaction reverts immediately.

6. No merging/splitting logic is reached.


### Impact

* mergeTVS() always reverts

* splitTVS() always reverts

No user can restructure vesting flows

This is a high-severity functional failure making two major features of the platform unusable.


### PoC

_No response_

### Mitigation


Allocate the array before writing:

`uint256[] memory newAmounts = new uint256[](length);`
  