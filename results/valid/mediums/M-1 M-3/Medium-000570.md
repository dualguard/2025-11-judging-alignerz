# [000570] # Stale State in `allocationOf` Mapping After NFT Claim or Burn Causes Incorrect State Read
  
  

## Summary

The failure to clear or update the `allocationOf` mapping after TVS claims or NFTs burns in `claimTokens` and `_merge` will cause incorrect state reads for users and external contracts as they will query stale allocation data for burned NFTs, leading to wrong calculations and decisions.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:971`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L970), when all flows are fully claimed, the NFT is burned via `nftContract.burn(nftId)` but `allocationOf[nftId]` is not cleared. Similarly, in `AlignerzVesting.sol:1048`, `_merge` burns the NFT but does not clear `allocationOf[nftId]`. Also, `allocationOf` is not updated when tokens are partly claimed or merged. Since `allocationOf` is a public mapping (line 114) accessible to external contracts and users, burned or updated NFTs retain stale allocation data, causing incorrect reads and calculations.

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

- An NFT must be claimed in `claimTokens`, OR
- An NFT must be merged into another TVS in `mergeTVS` (which calls `_merge`)
- The affected NFT's token ID must be queried via the `allocationOf` mapping after the burn

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User claims tokens from an NFT by calling `claimTokens`
2. `claimTokens` burns the NFT at line 971 or increases the TVS claimedSeconds but does not update `allocationOf[nftId]`
3. External contract (e.g., `A26ZDividendDistributor`) or user queries `allocationOf(nftId)` for the burned NFT
4. The query returns stale allocation data (amounts, vesting periods, claimed status) as if the NFT still exists
5. The external contract performs calculations based on this stale data, resulting in incorrect results
6. Alternatively, a user merges an NFT via `mergeTVS`, which calls `_merge` and burns the NFT at line 1048 without clearing `allocationOf[nftId]`
7. Queries for the burned NFT's allocation return outdated information, causing confusion or incorrect behavior

&nbsp;

## Impact

Users and external contracts reading `allocationOf` for claimed or burned NFTs receive stale data, leading to incorrect calculations and decisions. 


&nbsp;

## Proof of Concept
N/A


&nbsp;

## Mitigation

Update the `allocationOf` mapping when NFTs tempered in `claimTokens` or `mergeTVS`.

  