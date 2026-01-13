# [000572] Stale Allocation Data in `biddingProjects` Mapping After NFT Burn Causes Incorrect State Reads
  
  
&nbsp;

## Summary

The failure to clear allocation data from `biddingProjects[projectId].allocations[nftId]` after NFTs are burned in `claimTokens` and `_merge` will cause incorrect state reads for users and external contracts as they will access stale allocation data for burned NFTs, leading to wrong calculations and decisions.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:971`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L970), when all flows are fully claimed, the NFT is burned via `nftContract.burn(nftId)` but `biddingProjects[projectId].allocations[nftId]` is not cleared. Similarly, in `AlignerzVesting.sol:1048`, `_merge` burns the NFT but does not clear `biddingProjects[projectId].allocations[nftId]`. Since `biddingProjects` is a public mapping (line 99) and its nested `allocations` mapping (line 48) stores allocation data by NFT ID, burned NFTs retain stale allocation data, causing incorrect reads and calculations.

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

- An NFT from a bidding project must be fully claimed (all flows completed) in `claimTokens`, OR
- An NFT from a bidding project must be merged into another TVS in `mergeTVS` (which calls `_merge`)
- The burned NFT's allocation must be queried via `biddingProjects[projectId].allocations[nftId]` after the burn

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User fully claims all tokens from a bidding project NFT by calling `claimTokens(projectId, nftId)`
2. `claimTokens` accesses `biddingProjects[projectId].allocations[nftId]` at line 947 and burns the NFT at line 971, but does not clear the allocation data
3. External contract or user queries `biddingProjects[projectId].allocations[nftId]` for the burned NFT
4. The query returns stale allocation data (amounts, vesting periods, claimed status, assignedPoolId) as if the NFT still exists
5. The external contract performs calculations based on this stale data, resulting in incorrect results
6. Alternatively, a user merges a bidding project NFT via `mergeTVS`, which calls `_merge` and accesses `biddingProjects[projectId].allocations[nftId]` at line 1034
7. `_merge` burns the NFT at line 1048 without clearing `biddingProjects[projectId].allocations[nftId]`
8. Queries for the burned NFT's allocation return outdated information, causing confusion or incorrect behavior

&nbsp;

## Impact

Users and external contracts reading `biddingProjects[projectId].allocations[nftId]` for burned NFTs receive stale data, leading to incorrect calculations and decisions. 

&nbsp;

## Proof of Concept

N/A

&nbsp;

## Mitigation

Clear the allocation data from `biddingProjects` when NFTs are burned. 
  