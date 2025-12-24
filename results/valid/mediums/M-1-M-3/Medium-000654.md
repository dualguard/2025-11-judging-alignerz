# [000654] Stale allocationOf Mapping Enables Dividend Theft via Incorrect Unclaimed TVS Calculation
  
  ### Summary

The desynchronization between the authoritative project allocations and the public `allocationOf` mapping will cause dividend overpayment to users who have partially claimed their vesting as `A26ZDividendDistributor` will read stale `claimedSeconds` and `amounts` data from `allocationOf`, treating already-withdrawn tokens as still locked when calculating dividend weights.

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L571C12-L571C68

In `AlignerzVesting.sol`, the `allocationOf[nftId]` mapping is written during NFT creation (`claimNFT()`, `_claimRewardTVS()`, `distributeRemainingRewardTVS()`, and `splitTVS()`), but is never updated when:
- `claimTokens()` modifies `biddingProjects[projectId].allocations[nftId].claimedSeconds` and `claimedFlows` to reflect vesting withdrawals
- `mergeTVS()` modifies `allocations[mergedNftId].amounts`, `claimedSeconds`, and other fields to consolidate multiple NFTs

This creates a permanent desync where `allocationOf[nftId]` retains stale vesting state while the authoritative `allocations[nftId]` in project storage reflects the true current state.

`A26ZDividendDistributor.getUnclaimedAmounts()` queries only `vesting.allocationOf(nftId)` via the `IAlignerzVesting` interface to compute each NFT's unclaimed token value, using the stale `claimedSeconds` to calculate
When `allocationOf[nftId].claimedSeconds[i]` is 0 (or outdated), the distributor treats the full original `amounts[i]` as unclaimed, even if the user has already withdrawn a significant portion via `claimTokens()`.


### Internal Pre-conditions

1. User needs to receive a TVS NFT via `claimNFT()` or `_claimRewardTVS()` to initialize `allocationOf[nftId]` with the initial allocation state
2. User needs to call `claimTokens()` at least once to withdraw partial vesting, which updates `biddingProjects[projectId].allocations[nftId].claimedSeconds` but leaves `allocationOf[nftId].claimedSeconds` at the old value
3. Owner needs to call `setUpTheDividends()` to trigger dividend calculation after users have partially claimed



### External Pre-conditions

none

### Attack Path

1. User receives a TVS NFT with 10,000 tokens locked via `claimNFT()` at time T0
2. At this point, both `biddingProjects[projectId].allocations[nftId].claimedSeconds[0]` and `allocationOf[nftId].claimedSeconds[0]` are set to 0
3. User waits until 50% of the vesting period elapses
4. User calls `claimTokens(projectId, nftId)` and withdraws 5,000 tokens (50% vested)
5. The contract updates `biddingProjects[projectId].allocations[nftId].claimedSeconds[0]` to `vestingPeriod / 2`
6. However, `allocationOf[nftId].claimedSeconds[0]` remains 0 because `claimTokens()` never writes to this mapping
7. Owner calls `setUpTheDividends()` to distribute 1,000 USDC in dividends
8. `A26ZDividendDistributor.getUnclaimedAmounts(nftId)` reads `vesting.allocationOf(nftId).claimedSeconds[0]` and sees 0
9. The distributor calculates unclaimed tokens as `10,000 - (0 * 10,000 / vestingPeriod) = 10,000` instead of the actual 5,000 still locked
10. The distributor weights this user's dividend share as if they have 10,000 locked tokens
11. If the total unclaimed pool is computed as 20,000 tokens (including this user's inflated 10,000), the user receives `1,000 * 10,000 / 20,000 = 500 USDC` instead of the correct `1,000 * 5,000 / 15,000 = 333 USDC`
12. User gains 167 USDC excess dividends in this round at the expense of honest participants
13. User repeats this exploitation across multiple dividend rounds until full vesting, accumulating excess dividends each time


### Impact

Honest TVS holders who have not yet claimed their vesting (or whose `allocationOf` happens to be synchronized) suffer a proportional loss of dividend allocation as the dividend pool is incorrectly weighted toward users with stale `allocationOf` data. In a scenario where 50% of the user base has claimed 50% of their tokens but `allocationOf` still shows them as fully unclaimed, honest users receive approximately 33% less dividends than entitled. The exploiting users gain this excess dividend allocation (potentially hundreds to thousands of USDC per round depending on `stablecoinAmountToDistribute`) without any additional cost or risk, effectively stealing from the collective dividend pool that should be distributed fairly based on actual locked TVS.

### PoC

_No response_

### Mitigation

_No response_
  