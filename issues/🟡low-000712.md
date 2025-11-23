# [000712] More than 10 pools can be created due to off-by-one guard
  
  
## summary
createPool permits creation when `poolCount == 10`, allowing an 11th pool after increment.

## Finding Description
The precondition uses `<= 10` prior to incrementing `poolCount`. Because `poolCount` is increased after assignment, the 11th pool passes the guard.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L679-L701
[affected function code
](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L688-L689)
```solidity
require(!biddingProject.closed, Project_Already_Closed());
require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
...
biddingProject.poolCount++;
```
Root cause: off-by-one check with `<= 10` instead of `< 10` for a cap of 10 pools. Highest-impact scenario: admin can unintentionally create an 11th pool, breaking off-chain assumptions and allocation tooling expecting a hard cap of 10.

Location:
- `protocol/src/contracts/vesting/AlignerzVesting.sol:689`

## Impact
Medium. Configuration exceeds stated cap, potentially breaking off-chain processes.

## Recommendation
Use `< 10` to enforce a strict cap.

```diff
- require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project());
+ require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project());
```

  