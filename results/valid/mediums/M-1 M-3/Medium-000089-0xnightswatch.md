# [000089] Stale allocationOf mapping breaks dividend distribution after token claims and merges
  
  ### Summary

The failure to update `allocationOf[nftId]` in `claimTokens` and `mergeTVS` will cause incorrect dividend calculations for NFT holders as users claim vested tokens or merge NFTs, since the `A26ZDividendDistributor` contract reads stale allocation data that doesn't reflect claimed amounts or merged allocations.

### Root Cause

In [AlignerzVesting.sol:941-975](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975), the `claimTokens()`updates the allocation's `claimedSeconds()` and `claimedFlows()` arrays in project-specific storage but never syncs these changes to the global `allocationOf[nftId]` mapping.  Similarly, in [AlignerzVesting.sol:1002-1026](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026), the `mergeTVS()`  modifies the merged allocation but doesn't update `allocationOf[mergedNftId]` after the merge completes. 

### Internal Pre-conditions

1. User needs to mint an NFT via `claimNFT()` or `claimRewardTVS()`
2. User needs to call `claimTokens()` to claim vested tokens OR call `mergeTVS()` to merge multiple NFTs
3. The A26ZDividendDistributor contract needs to call `getUnclaimedAmounts()` to calculate dividends

### External Pre-conditions

None

### Attack Path


1. User mints NFT with allocation (e.g., 1000 tokens vesting over 365 days)
   - allocationOf[nftId] is set with claimedSeconds = [0] (AlignerzVesting.sol:887)

2. After 180 days, user calls claimTokens(), claiming ~493 tokens
   - Project-specific allocation updated: claimedSeconds = [180 days] (AlignerzVesting.sol:959-962)
   - allocationOf[nftId] remains unchanged: claimedSeconds = [0] (stale)

3. A26ZDividendDistributor calls getUnclaimedAmounts(nftId)
   - Reads allocationOf[nftId] (A26ZDividendDistributor.sol:142-143)
   - Uses stale claimedSeconds = [0], incorrectly calculating 1000 unclaimed tokens instead of ~507

4. Result: user receives double the dividends they should get

Same issue occurs with mergeTVS
   - Merged allocation is updated but allocationOf[mergedNftId] is never updated (AlignerzVesting.sol:1002-1026)


### Impact

User who claimed tokens or merge NFTs will have their dividend allocations calcualted incorrectly based on stale data. This cause:

- Unfair dividend distribution: Users who claimed tokens appear to have more unclaimed tokens than they acctually do, recieving excess dividends.
- Protocol revenue loss: The dividend pool is distributed  incorrectly, with some user getting more than their fair share.
- Inconsistent state: The global `allocationOf` mapping diverges from the actual project-specific allocations. 

### PoC

A coded PoC demonstrating this issue requires significant setup involving merkle tree generation, bidding workflows, and NFT minting. Due to the time-consuming nature of properly configuring the test environment with valid merkle proofs and project state, the PoC is omitted. However, the bug is evident from code inspection: `claimTokens` updates the project-specific allocation storage but never syncs these changes to the global `allocationOf` mapping that was set during NFT minting.

### Mitigation

Add allocationOf update to both fucntions: 

In `claimTokens()`:
```diff
if (flowsClaimed == nbOfFlows) {  
    nftContract.burn(nftId);  
    allocation.isClaimed = true;  
}  
token.safeTransfer(msg.sender, claimableAmounts);  
  
+// Add this line to sync the global mapping  
+allocationOf[nftId] = allocation;  
  
emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed
```

In  `mergeTVS()`: 
```diff
token.safeTransfer(treasury, feeAmount);  
  
+// Add this line to sync the global mapping  
+allocationOf[mergedNftId] = mergedTVS;  
  
emit TVSsMerged(projectId, isBiddingProject, nftIds, mergedNftId, mergedTVS.amounts, mergedTVS.vestingPeriods, mergedTVS.vestingStartTimes, mergedTVS.claimedSeconds, mergedTVS.claimedFlows);
```
  