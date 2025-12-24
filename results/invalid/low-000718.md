# [000718] Pool count validation allows creation of 11 pools instead of maximum 10
  
  The `createPool` function checks `require(biddingProject.poolCount <= 10)` before creating a new pool. Since `poolCount` starts at 0, this allows pools to be created when `poolCount` is 0 through 10, resulting in 11 total pools (IDs 0-10) instead of the intended maximum of 10 pools. The check should use `<` instead of `<=` to enforce the correct limit.

## Recommendation
Change the validation to use strict less-than comparison:

```solidity
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

  