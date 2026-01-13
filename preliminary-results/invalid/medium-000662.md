# [000662] Missing storage `__gap` in AlignerzVesting upgradeable contract
  
  ### Summary

AlignerzVesting is an upgradeable contract (using UUPS pattern) but does not declare a storage gap. While its parent contracts, like `WhitelistManager`, include gaps (`uint256[50] private __gap;`), the absence of a gap in `AlignerzVesting` introduces a potential risk for future upgrades. 
Adding new state variables in upgrades could accidentally overwrite storage slots of existing variables, leading to storage collisions.

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol
`AlignerzVesting` inherits from upgradeable contracts but does not include its own storage gap. This omission leaves no reserved slots for future state variables, which violates the best practices for upgradeable contracts.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

/

### Impact

Future upgrades that add new state variables to `AlignerzVesting` contract may corrupt existing storage, causing unpredictable behavior, loss of data, or contract malfunction. 

### PoC

_No response_

### Mitigation

Add a reserved storage `__gap` in `AlignerzVesting.sol`, for example:
```solidity
uint256[50] private __gap;
```
This ensures future variables can be safely added without overwriting existing storage.

  