# [000626] Owner will create 11 pools instead of 10 maximum pools
  
  ### Summary

Owner will create 11 pools instead of 10 maximum pools violating the intended pool limit

### Root Cause

in [createPool](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689) function the validation uses <= instead of <
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
The requirement is here to not exceed 10 pools per project, as the error mentions

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Owner calls createPool 11 times

### Impact

The protocol allows one more pool than intended (11 instead of 10)

### PoC

```diff
-require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

### Mitigation

_No response_
  