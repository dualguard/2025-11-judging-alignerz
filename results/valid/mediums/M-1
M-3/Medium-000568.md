# [000568] Stale Data Synchronization Bug in `allocationOf` Mapping Causes Incorrect Dividend Distribution and Enables Perpetual Dividend Exploitation
  
  

## Summary

The failure to synchronize the `allocationOf` mapping after token claims and merge operations will cause incorrect dividend calculations and enable perpetual dividend exploitation for users, as the `A26ZDividendDistributor` contract reads stale allocation data that does not reflect claimed tokens, allowing attackers to claim nearly all tokens while maintaining eligibility for dividends on already-claimed amounts.

```js
function claimTokens(...) external {...}

function mergeTVS(...) external returns(uint256){}
```
&nbsp;

## Root Cause

In [`AlignerzVesting.sol:946-975`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975), the `claimTokens()` function updates only the storage allocation in `biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]`, but does not update the `allocationOf[nftId]` mapping. The `allocationOf` mapping is only set during NFT creation (lines 571, 614, 888, 1093) and becomes stale after any claim operation. Similarly, in [`AlignerzVesting.sol:1003-1026`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026), the `mergeTVS()` function updates `mergedTVS` but does not update `allocationOf[mergedNftId]` after merging flows. When `A26ZDividendDistributor::getUnclaimedAmounts()` (line 140-161) reads from `allocationOf[nftId]`, it receives outdated values for `claimedSeconds`, `claimedFlows`, and `amounts`, causing incorrect calculation of unclaimed amounts and subsequent dividend distribution.

&nbsp;

## Internal Pre-conditions

1. A user needs to own an NFT representing a TVS with at least one unclaimed flow
2. The user needs to call `claimTokens()` to claim tokens, which updates the storage allocation but not `allocationOf`
3. The owner of `A26ZDividendDistributor` needs to call `setUpTheDividends()` or `setAmounts()` followed by `setDividends()` to trigger dividend distribution calculations
4. For the exploitation scenario, a user needs to claim tokens such that at least one flow remains unclaimed (leaving at least 1 wei unclaimed) to prevent NFT burning

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. A user owns an NFT with a TVS containing multiple flows with significant token amounts
2. The user calls `claimTokens()` to claim most of their tokens, leaving only a minimal amount (e.g., 1 wei) unclaimed in one flow to prevent the NFT from being burned
3. The `claimTokens()` function updates `biddingProjects[projectId].allocations[nftId].claimedSeconds` and `claimedFlows`, but does not update `allocationOf[nftId]`
4. The owner of `A26ZDividendDistributor` calls `setUpTheDividends()`, which internally calls `getTotalUnclaimedAmounts()`
5. `getTotalUnclaimedAmounts()` iterates through all NFTs and calls `getUnclaimedAmounts(nftId)` for each owned NFT
6. `getUnclaimedAmounts()` reads stale data from `allocationOf[nftId]`, which still shows the original unclaimed amounts and zero `claimedSeconds`
7. The function calculates unclaimed amounts based on stale data, including tokens that have already been claimed
8. `_setDividends()` distributes dividends proportionally based on these incorrect unclaimed amounts
9. The attacker receives dividends for tokens they have already claimed, and can repeat this process in subsequent dividend distribution cycles

&nbsp;

## Impact

Users who have claimed tokens enjoy incorrect dividend calculations, receiving dividends for already-claimed tokens. Attackers can exploit this by strategically claiming nearly all tokens while leaving a minimal amount unclaimed to keep the NFT alive, then perpetually receiving dividends based on stale allocation data. The protocol suffers a loss proportional to the incorrectly calculated dividends distributed to users with stale `allocationOf` data. Additionally, users who have merged TVSs also have stale `allocationOf` data, further compounding the issue.

&nbsp;

## Proof of Concept

N/A

&nbsp;

## Mitigation

Update the `allocationOf` mapping whenever the underlying allocation storage is modified. Add the following synchronization after state updates:

1. In `claimTokens()` (after line 973), add:
```solidity
allocationOf[nftId] = allocation;
```

2. In `mergeTVS()` (after line 1015 and after the merge loop completes), add:
```solidity
allocationOf[mergedNftId] = mergedTVS;
```

3. Consider creating a helper function to ensure synchronization:
```solidity
function _syncAllocationOf(uint256 nftId, Allocation storage allocation) internal {
    allocationOf[nftId] = allocation;
}
```

  