# [000240] Pool Count Limit Allows Creation of 11 Pools Instead of Intended Maximum of 10
  
  - **Description**

The `AlignerzVesting::createPool()` function uses a less-than-or-equal comparison (`<=`). 

- Pool IDs are 0-indexed: 0, 1, 2, ..., 9 = 10 pools
- When `poolCount = 0`, creates pool 0, then increments to 1
- When `poolCount = 9`, creates pool 9, then increments to 10
- When `poolCount = 10`, the check `10 <= 10` passes, creates pool 10 (the 11th pool), then increments to 11

- **Impact**

All functionality designed for 10 pools could potentially malfunction.

- **Recommended Mitigation**

Change the comparison operator to strictly less-than:

```solidity
require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```
  