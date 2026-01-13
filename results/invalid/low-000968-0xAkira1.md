# [000968] Missing Validation in `setTVSAllocation` and `setStablecoinAllocation`
  
  ### Summary

The `setTVSAllocation` and `setStablecoinAllocation` functions lacks validation checks for project existence, initialization status, and deadline status. This allows the owner to accidentally allocate tokens to non-existent projects, uninitialized projects, or projects that have already passed their claim deadline, temporarily locking funds until manual recovery via withdrawStuckTokens.

### Root Cause

The `setTVSAllocation` and `setStablecoinAllocation` functions does not check the critical state of the project before processing allocations:
No project existence check: the function does not check that `rewardProjectId < rewardProjectCount`, as in other functions. 

No deadline check: the function does not check that `block.timestamp < rewardProject.claimDeadline`, which allows funds to be allocated to projects in which KOLs can no longer request tokens using the regular `claimRewardTVS` function.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The owner must call this function with invalid parameters. 

### Impact

Due to the mistake, the owner will set new values in the protocol and transfer the tokens to the contract. When he notices the mistake, he will be able to retrieve the stuck tokens and reset the new values.

Due to these actions, the owner will spend more native tokens on gas than the check will interrupt his actions. 

### PoC

_No response_

### Mitigation

Add appropriate checks to prevent accidentally sending tokens to the contract. 
`rewardProjectId < rewardProjectCount`
  