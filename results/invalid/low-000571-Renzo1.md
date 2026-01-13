# [000571] Stale Allocation Data in `rewardProjects` Mapping After NFT Burn Causes Incorrect State Reads
  
  

## Summary

The failure to clear allocation data from `rewardProjects[projectId].allocations[nftId]` after NFTs are burned in `claimTokens` and `_merge` will cause incorrect state reads for users and external contracts as they will access stale allocation data for burned NFTs, leading to wrong calculations and decisions.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:971`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L970), when all flows are fully claimed, the NFT is burned via `nftContract.burn(nftId)` but `rewardProjects[projectId].allocations[nftId]` is not cleared. Similarly, in `AlignerzVesting.sol:1048`, `_merge` burns the NFT but does not clear `rewardProjects[projectId].allocations[nftId]`. Since `rewardProjects` is a public mapping (line 102) and its nested `allocations` mapping (line 28) stores allocation data by NFT ID, burned NFTs retain stale allocation data, causing incorrect reads and calculations.

```js
    function claimTokens(uint256 projectId, uint256 nftId) external {
		// ...
        if (flowsClaimed == nbOfFlows) {
            nftContract.burn(nftId);
            allocation.isClaimed = true;
        }
		// ...
	}
```

&nbsp;

## Internal Pre-conditions

- An NFT from a reward project must be fully claimed (all flows completed) in `claimTokens`, OR
- An NFT from a reward project must be merged into another TVS in `mergeTVS` (which calls `_merge`)
- The burned NFT's allocation must be queried via `rewardProjects[projectId].allocations[nftId]` after the burn

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User fully claims all tokens from a reward project NFT by calling `claimTokens(projectId, nftId)`
2. `claimTokens` accesses `rewardProjects[projectId].allocations[nftId]` at line 948 and burns the NFT at line 971, but does not clear the allocation data
3. External contract or user queries `rewardProjects[projectId].allocations[nftId]` for the burned NFT
4. The query returns stale allocation data (amounts, vesting periods, claimed status) as if the NFT still exists
5. The external contract performs calculations based on this stale data, resulting in incorrect results
6. Alternatively, a user merges a reward project NFT via `mergeTVS`, which calls `_merge` and accesses `rewardProjects[projectId].allocations[nftId]` at line 1035
7. `_merge` burns the NFT at line 1048 without clearing `rewardProjects[projectId].allocations[nftId]`
8. Queries for the burned NFT's allocation return outdated information, causing confusion or incorrect behavior

&nbsp;

## Impact

Users and external contracts reading `rewardProjects[projectId].allocations[nftId]` for burned NFTs receive stale data, leading to incorrect calculations and decisions. Since `allocationOf` also references these allocations (lines 571, 614), the same burned NFT can return stale data through both paths, compounding the issue and potentially causing financial discrepancies in dependent systems.

&nbsp;

## Proof of Concept

N/A

&nbsp;

## Mitigation

Clear the allocation data from `rewardProjects` when NFTs are burned. 
  