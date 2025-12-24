# [000088] Users overpay fees on already-claimed tokens causing protocol insolvency
  
  ### Summary

The fee calculation logic in `mergeTVS`, `_merge`, and `splitTVS` charges fees on original allocation amounts without accounting for already-claimed tokens, causing users to overpay fees and creating protocol insolvency as the treasury collects fees on tokens that were already distributed to users.

### Root Cause

In `AlignerzVesting.sol`, the fee calculation uses amounts[i] (the original full allocation) instead of calculating the remaining unclaimed amount as amounts[i] - (amounts[i] * claimedSeconds[i] / vestingPeriods[i]).

In `mergeTVS()`: The function calls `calculateFeeAndNewAmountForOneTVS()` which charges fees on the full `amounts` array without considering claimedSeconds. [AlignerzVesting.sol:1011-1014](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1011-L1014)

In `_merge()`: For each flow being merged, it calculates `fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j])` using the full original amount, ignoring `TVSToMerge.claimedSeconds[j]`. [AlignerzVesting.sol:1037-1040](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1037-L1040)

In `splitTVS()`: Similar to `mergeTVS()`, it calls `calculateFeeAndNewAmountForOneTVS()` on the full amounts array without accounting for claimed portions. [AlignerzVesting.sol:1067-1070](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1067-L1070)

### Internal Pre-conditions

1. User must have an NFT with vested tokens where `claimedSeconds[i] > 0` (tokens have been partially or fully claimed)
2. User calls `mergeTVS()` or `splitTVS()` on this NFT
3. Fee rates (`mergeFeeRate` or `splitFeeRate`) must be greater than 0

### External Pre-conditions

None

### Attack Path

This is not an attack but a systemic accounting error that occurs during normal protocol operations:

1. User claims vested tokens via `claimTokens()`, updating `allocation.claimedSeconds[i]` to reflect claimed time.

2. User later calls `mergeTVS()` or `splitTVS()` on their NFT.

3. Fee calculation uses `amounts[i]` (original allocation) instead of remaining unclaimed tokens:
   - Correct calculation should be: `amounts[i] - (amounts[i] * claimedSeconds[i] / vestingPeriods[i])`

4. Consequences:
   - User pays fees on tokens they already claimed and no longer own.
   - Treasury receives fees for already-distributed tokens, creating an accounting deficit.
   - Protocol can become insolvent as treasury balance no longer matches expected fee revenue.

### Impact

User suffer financial loss by by overpaying fess on tokens they've already claimed and no longer own

Protocol becomes insolvent as the treasury collects fees on non-existents tokens, creating a mismatch between expected and actual token balances. 

### PoC

N/A

### Mitigation

Fix1: Update `calculateFeeAndNewAmountForOneTVS()` to calcualte fee on the remaining unclaimed amounts.

Fix2: Update `_merge()` to calculate fees on the remaining amounts

Fix3: Update all call sites to pass teh additional parameter `claimedSeconds` and `vestingPeriods` to the fee calculation
  