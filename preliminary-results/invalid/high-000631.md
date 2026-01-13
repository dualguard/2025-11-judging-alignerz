# [000631] Users will permanently lock vested tokens of TVS holders by self-merging their own NFT
  
  ### Summary

The lack of validation preventing an `NFT `from being merged into itself in `AlignerzVesting.mergeTVS ` will cause a permanent loss of claimability for `TVS` holders as  a user will include the destination `NFT `inside the `nftIds` array, triggering a burn of the only remaining vesting certificate.

### Root Cause

In `AlignerzVesting.sol:1002 (mergeTVS) `  
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002

and `_merge` at `AlignerzVesting.sol:1028`
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1028
, there is no explicit check ensuring `mergedNftId != nftId `for each element in `nftIds`. The loop unconditionally burns each `nftId`, even if it is the same as the target `mergedNftId`. If the same NFT is both the merge target and a merge source:

- Fees are (attempted to be) applied to its own allocation.
- `_merge` burns the NFT supplying the flows.
- The contract continues as if the merged target still exists.

- Result: The only NFT tracking the vesting flows is destroyed; its underlying token amounts become permanently inaccessible.

### Internal Pre-conditions

- Vested user must own an NFT representing at least one active vesting flow.
- User must know the mergedNftId (their own NFT ID).
- `mergeFeeRate`  may be zero or non-zero (fee doesn’t block the logic).
- `nftIds.length`>= 1.
- Storage for that `NFT ` has non-empty amounts.

### External Pre-conditions

None

### Attack Path

- User claims or acquires a vesting NFT (e.g., via c`laimNFT` or `claimRewardTVS`).
- User calls `mergeTVS`(`projectId`, `mergedNftId`, [projectId], [mergedNftId]) where mergedNftId is included in nftIds.
- `_merge` executes and burns `mergedNftId `inside its own merge cycle.
- Control returns; function emits `TVSsMerged` suggesting success.
- Subsequent` claimTokens(projectId, mergedNftId)` call reverts because the NFT no longer exists—tokens stranded

### Impact

The affected party (vested user) suffers a permanent loss of 100% of the remaining claimable vested tokens tied to that NFT. The protocol also accrues unclaimable stranded token balance increasing accounting inconsistency. User doesn’t gain funds directly but can grief themselves. Severity is HIGH due to irreversible token lock.

### PoC

- Own `NFT ID` = 12.
- Call `mergeTVS(projectId, 12, [projectId], [12])`.
-` _merge` burns `NFT #12`.
- Attempt `claimTokens(projectId, 12)` → fails (no ownership / burned).
- Tokens permanently locked.

### Mitigation

- Add validation in mergeTVS: for each `nftId -> require(nftId != mergedNftId, "Cannot self-merge");`.
  