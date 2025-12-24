# [000399] Anyone can call distributeRewardTVS() missing access control
  
  ### Summary

The function `distributeRewardTVS()` is intended to be executed only by the pool/project owner to force-claim unclaimed TVS allocations after the claim deadline has passed. Its purpose is to ensure that leftover TVS tokens are properly distributed to the KOLs who never claimed their rewards on time.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L522?plain=1
However, the function lacks any access control. Despite the comment stating:
```soldity
“Allows the owner to distribute the TVS that have not been claimed yet…”
```

the code declares the function as:
```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external
```

with no onlyOwner check or any equivalent authorization. This allows any external account to call this function and trigger forced reward claims.

### Root Cause

- Missing onlyOwner / access-control check on `distributeRewardTVS()`.
- Developer intention (per comment) does not match implementation.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

Control of TVS distribution is not restricted to the project owner, exposing the protocol to unauthorized state mutations and reward distribution corruption.

### PoC

_No response_

### Mitigation

- Add proper access control, e.g.:
```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol)
    external
    onlyProjectOwner(rewardProjectId)
{ ... }
```
  