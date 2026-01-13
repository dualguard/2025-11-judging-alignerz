# [000522] Incorrect projectId Validation in claimTokens Enables Cross-Project Token Theft
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L941
The `claimTokens` function relies on a user-provided `projectId` to access project-specific allocations but validates ownership only via the global NFT contract. This allows users to claim and drain tokens from unrelated projects by specifying a mismatched `projectId` while using their own NFT ID, potentially burning the wrong NFT and stealing tokens.


### Root Cause

The function fetches the allocation from `biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]` without verifying that the `nftId` belongs to the specified `projectId`. NFT IDs are global and can collide across projects.


### Internal Pre-conditions

- Multiple projects exist in the contract.
- An allocation entry exists for the same `nftId` in a different project (can be uninitialized or leftover).


### External Pre-conditions

- Attacker owns an NFT from one project.
- Victim project has tokens vested and allocations mapped.


### Attack Path

No response

### Impact

Full drainage of vested tokens from any project, leading to permanent loss for legitimate users.


### PoC



No response

### Mitigation

Store `projectId` and `isBiddingProject` per NFT in a global mapping. Validate in `claimTokens`:
```diff
+ require(NFTProject[nftId] == projectId && NFTIsBidding[nftId] == isBiddingProject, "Mismatch");
```
  