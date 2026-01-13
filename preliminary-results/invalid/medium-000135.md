# [000135] The protocol fails to enforce the number of pools allowed correctly
  
  ### Summary

`createPool` enforces `require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());`. Because `poolCount` starts at 0 and increments after pool creation, this condition still passes when `poolCount == 10`, letting a project end up with 11 pools (`AlignerzVesting.sol`, `createPool`).  

### Root Cause

.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- 10 pools already exist
- `createPool` is called
- `require(biddingProject.poolCount <= 10` doesn't revert as pool count is still 10
- 11 pools exist

### Impact

Protocol invariants around pool caps are broken; an extra pool can be launched even after the project already has the intended maximum. This breaks an invariant that the protocol wants to enforce. 

### PoC

.

### Mitigation

Change the guard to `require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());` so the 11th creation attempt reverts before minting/importing tokens. Suggest adding a test to assert that creating an 11th pool fails.
  