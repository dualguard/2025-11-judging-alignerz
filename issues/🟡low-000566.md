# [000566] Excessive Fee Charging on Destination NFT During Merge Causes Unfair Double Fee Extraction from Users
  
  
## Summary

Charging merge fees on both the destination NFT's existing amounts and the NFTs being merged into it will cause excessive fee extraction for users merging TVSs as the protocol charges fees on tokens it does not process, resulting in users paying fees twice on their own tokens.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:1014`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013-L1014), the `mergeTVS` function charges merge fees on the `mergedNftId`'s existing amounts before merging other NFTs into it. Then in `AlignerzVesting.sol:1040`, the `_merge` function charges fees again on each NFT being merged. Chargng fees on the destination NFT (`mergedNftId`) is a problmatic because those tokens are already owned by the user and the protocol does not process, move, or modify them during the merge operation. The fee should only apply to the NFTs being merged (burned), not to the destination NFT that simply receives the merged tokens.

&nbsp;

## Internal Pre-conditions

1. User needs to own a TVS NFT with `mergedNftId` to use as the destination for merging
2. User needs to own at least one other TVS NFT in `nftIds` to merge into `mergedNftId`
3. The `mergedNftId` must have a non-zero `allocation.amounts` array to trigger the fee calculation at line 1014
4. `mergeFeeRate` must be greater than 0 to charge fees

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. User calls `mergeTVS(projectId, mergedNftId, projectIds, nftIds)` to merge multiple NFTs into an existing NFT
2. Function retrieves the `mergedNftId` allocation at lines 1008-1010
3. Function calculates and charges merge fee on `mergedNftId`'s existing amounts at line 1014, reducing those amounts in storage at line 1015
4. Function loops through `nftIds` and calls `_merge` for each NFT being merged at line 1022
5. Inside `_merge`, function calculates and charges merge fee on each NFT being merged at line 1040
6. Function transfers the total accumulated fees (from both `mergedNftId` and all merged NFTs) to the treasury at line 1024
7. User loses tokens from both the destination NFT and the NFTs being merged, paying fees on tokens they already own

&nbsp;

## Impact

Users suffer excessive fee extraction when merging TVSs. They pay merge fees on both the destination NFT's existing tokens (which the protocol does not process) and the NFTs being merged into it. 


&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Remove the fee calculation and amount reduction for the `mergedNftId` at lines 1012-1015. Only charge merge fees on the NFTs being merged (which is already handled in `_merge` at line 1040). The destination NFT should retain its original amounts without any fee deduction, as the protocol does not process those tokens during the merge operation. 
  