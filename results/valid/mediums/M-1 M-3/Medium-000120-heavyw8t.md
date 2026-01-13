# [000120] AllocationOf not updated on merge and claims leads to stale vesting data for dividend distributor
  
  ### Summary

`allocationOf[nftId]` in `AlignerzVesting` is a one-time snapshot that is never updated when users claim or merge TVSs, while `A26ZDividendDistributor` uses this stale snapshot as the sole source of truth to compute “unclaimed” amounts. This causes users who already claimed part or all of their TVS to still receive dividends as if they had the full original position. Severity is high because it directly overpays dividends to some users and underpays others.


### Root Cause

`allocationOf` is only written when NFTs are created/distributed/split, but all later state changes happen in per-project allocations only:
```solidity

mapping(uint256 => Allocation) public allocationOf; // snapshot

// Claims update only project-local storage
function claimTokens(uint256 projectId, uint256 nftId) external {
    (Allocation storage allocation, IERC20 token) = NFTBelongsToBiddingProject[nftId]
        ? (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    // mutate allocation.claimedSeconds / claimedFlows
    // ...
    // allocationOf[nftId] is never updated here
}
```
`A26ZDividendDistributor.getUnclaimedAmounts()` reads only `vesting.allocationOf(nftId)` and therefore always sees the old snapshot.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- Attacker obtains a large TVS and `allocationOf[nftId]` is recorded with a high `amounts[]` and zero `claimedSeconds`.
- Attacker calls `claimTokens()` multiple times to pull out as much TVS principal as possible; only project-local allocations are updated, `allocationOf[nftId]` stays unchanged.
- Protocol owner calls `setUpTheDividends();` `A26ZDividendDistributor` uses `vesting.allocationOf(nftId)` and counts the attacker as still having the original full allocation.
- Dividends are distributed; the attacker receives dividends proportional to the original large TVS, despite now holding only a much smaller (or zero) true unclaimed balance, while honest holders are underpaid.

### Impact

A user who has already claimed most or all of their TVS principal continues to be treated as holding the full, unclaimed amount when dividends are set up.

Dividend weights are computed from stale `allocationOf`, so that the user receives a larger share of the dividend pool than their true remaining TVS balance justifies.

Other users with accurate, unclaimed allocations receive proportionally less dividends, causing a direct, quantifiable misdistribution of stablecoin rewards.

### PoC

_No response_

### Mitigation

Consider removing the `allocationOf` snapshot entirely and having external consumers (like `A26ZDividendDistributor`) read the canonical per-project allocations via a dedicated view function, so they always operate on up‑to‑date vesting state. If `allocationOf` must be kept, ensure it is synchronously updated (or cleared) in every function that mutates allocations `(claimTokens(), _merge(), splits, burns)` so no stale snapshots remain.
  