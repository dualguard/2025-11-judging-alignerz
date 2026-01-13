# [000573] Stale Project Type Flag in `NFTBelongsToBiddingProject` Mapping After NFT Burn Causes Incorrect State Reads
  
  
## Summary

The failure to clear the `NFTBelongsToBiddingProject` mapping after NFTs are burned in `claimTokens` and `_merge` will cause incorrect state reads for users and external contracts as they will access stale project type information for burned NFTs, leading to wrong decisions and calculations.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:971`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L970),  when all flows are fully claimed, the NFT is burned via `nftContract.burn(nftId)` but `NFTBelongsToBiddingProject[nftId]` is not cleared. Similarly, in `AlignerzVesting.sol:1048`, `_merge` burns the NFT but does not clear `NFTBelongsToBiddingProject[nftId]`. Since `NFTBelongsToBiddingProject` is a public mapping (line 111) that tracks whether an NFT belongs to a bidding project (`true`) or reward project (`false`), burned NFTs retain stale project type flags, causing incorrect reads and decisions in systems that rely on this information.

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

- An NFT must be fully claimed (all flows completed) in `claimTokens`, OR
- An NFT must be merged into another TVS in `mergeTVS` (which calls `_merge`)
- The burned NFT's project type must be queried via the `NFTBelongsToBiddingProject` mapping after the burn

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User fully claims all tokens from an NFT by calling `claimTokens(projectId, nftId)`
2. `claimTokens` reads `NFTBelongsToBiddingProject[nftId]` at line 945 to determine which project type to access
3. `claimTokens` burns the NFT at line 971 but does not clear `NFTBelongsToBiddingProject[nftId]`
4. External contract or user queries `NFTBelongsToBiddingProject(nftId)` for the burned NFT
5. The query returns stale boolean value (`true` for bidding project, `false` for reward project) as if the NFT still exists
6. The external contract makes decisions based on this stale data, potentially routing to the wrong project type or making incorrect calculations
7. Alternatively, a user merges an NFT via `mergeTVS`, which calls `_merge` and reads `NFTBelongsToBiddingProject[nftId]` at line 1032
8. `_merge` burns the NFT at line 1048 without clearing `NFTBelongsToBiddingProject[nftId]`
9. Queries for the burned NFT's project type return outdated information, causing confusion or incorrect behavior in dependent systems

&nbsp;

## Impact

Users and external contracts reading `NFTBelongsToBiddingProject` for burned NFTs receive stale project type flags, leading to incorrect routing, calculations, and decisions. 


&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Clear the `NFTBelongsToBiddingProject` mapping when NFTs are burned.
  