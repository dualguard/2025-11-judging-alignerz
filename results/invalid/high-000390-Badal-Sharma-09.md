# [000390] Missing projectId validation in mergeTVS()
  
  ### Summary

`mergeTVS()` and its helper `_merge()` do not validate that the supplied `projectId` (and the `projectIds[]` array items) are valid project indices (i.e. `< biddingProjectCount` or `< rewardProjectCount`). Other functions (e.g. `placeBid`, `updateBid`) do this check, but `mergeTVS()` omits it. This allows callers to reference non-existent project slots or mix NFTs from different projects, leading to corrupted allocations, burned NFTs, and broken invariants.


### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002C5-L1005C5?plain=1
`mergeTVS()` and `_merge()` operate directly on `biddingProjects[projectId]` and `rewardProjects[projectId]`, and on `allocations[nftId]` without first verifying that `projectId` is within the valid range. The code assumes the passed project ids are valid and that the NFTs being merged belong to the expected project, but it never enforces that assumption.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Attacker (or benign user) calls `mergeTVS(projectId = X)` where `X >= biddingProjectCount`.
2. The contract loads `biddingProjects[X].allocations[mergedNftId]` (creates/reads uninitialized struct).
3. The merge appends flows from `nftIds[]` into that uninitialized storage, and burns the merged NFTs.
4. The allocations that used to be claimable under original project are now moved into an orphaned/unexpected storage slot. Users cannot find or claim them via normal project flow.

### Impact

- **Corrupt allocations / vesting data:** Merging into or reading from non-existent project storage may create or overwrite data in unexpected places. Merged flows could be appended to unintended allocation structs, corrupting vesting schedules.
- **Burn / data-loss risk:** Users can inadvertently or intentionally burn NFTs by passing incorrect project ids. Burned NFTs may no longer be claimable even though their allocations got orphaned.
- **Cross-project contamination:** NFTs from different projects (bidding vs reward, or different project IDs) can be merged together. This breaks the invariant that allocations belong to a specific project/token.

### PoC

Validate `projectId` at function entry:

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds)
    external returns (uint256)
{
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    // Continue...
}
```

### Mitigation

_No response_
  