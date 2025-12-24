# [000288] distributeStablecoinAllocation() Ignores The user's Passed Array Length Causing an Out-of-Bounds Element Retrieve
  
  ### Summary

(It's the exact same bug in [distributeRewardTVS()](https://github.com/dualguard/2025-11-alignerz-DunateIIo/issues/4) in a different root so I just copied & edited the previous report accordingly)
In `AlignerzVesting.sol`, inside `distributeStablecoinAllocation()`, the function takes an array of KOL addresses so anyone can call it after the deadline to distribute rewards. A KOL can pass only his own address to claim just his Reward. However, the function doesn’t iterate over the passed-in array — it iterates based on the state array’s length instead. This causes it to run the loop for the full stored array, not the provided one and revert.

### Root Cause

It should retrive the length of the input array instead of the main state array
https://github.com/dualguard/2025-11-alignerz-DunateIIo/blob/fe542df71d42e3a855f2b014032440ccc2b40da4/protocol/src/contracts/vesting/AlignerzVesting.sol#L543

Or just remove the parameter entirely and pull the addresses directly from the main array.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Best Case: The Caller passes the full list of KOL's addresses; **no Impact**
Regular Use: The caller tries to pass only his address, so **if** there is still more KOLs didn't claim their reward, **he'll lose the loop gas** for no outcome because it will revert.
Worst Case: The caller pass 99 address out of 100 KOLs who didn't claim the reward yet, `paying the gas of 99 loop` and revert on the 100th

### PoC

_No response_

### Mitigation

Make the iterations based on the passed array's length:

```
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        (...)
        uint256 len = kol.length;
        for (uint256 i; i < len;) {...
```
  