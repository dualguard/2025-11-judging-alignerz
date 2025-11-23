# [000353] [L-01] Unused import in AlignerzVesting.sol
  
  ### Summary

`IAlignerzVesting` interface is imported in `AlignerzVesting.sol` but never actually used.

### Root Cause

In `AlignerzVesting.sol:13` there is an import of `IAlignerzVesting` interface but this interface was not inherited or used

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Not clean code

### PoC

_No response_

### Mitigation

Remove all unused imports to improve code clarity and reduce compilation overhead.
This will help maintain cleaner code and prevent potential confusion during development.
  