# [000437] Access control is missing
  
  ### Summary


The functions `distributeRewardTVS` and `distributeStablecoinAllocation` allow the owner to distribute unclaimed TVS or stablecoin allocations to KOLs after the `claimDeadline` has passed. However, both functions are missing the necessary access control.


### Root Cause


```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
```

```solidity
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
```

According to the function NatSpec:

```
Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs

Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
```

Both functions are missing the `onlyOwner` modifier.


### Internal Pre-conditions

`claimDeadline` must pass

### External Pre-conditions

None

### Attack Path

See Root Cause

### Impact



This allows anyone to call `distributeRewardTVS` and `distributeStablecoinAllocation` after the `claimDeadline` has passed, bypassing the intended access control.


### PoC

NA

### Mitigation

implement `onlyOwner` modifier for `distributeStablecoinAllocation` and `distributeRewardTVS`.
  