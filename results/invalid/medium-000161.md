# [000161] Vesting pools amount caps at 11 pools rather than 10
  
  ### Summary

Incorrect check allows for an extra one.

### Root Cause

`createPool` is supposed to limit pool creation to 10, we know this because of the error inside the requirement block:
```solidity
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

However, the pool count is initialized as zero in `launchBiddingProject`, which means the pool creation does not revert at the 11th, only the 12th.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

-

### Impact

One extra pool than intended

### PoC

_No response_

### Mitigation

_No response_
  