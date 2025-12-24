# [000546] Fee Calculation on Full Flow Amount Instead of Unclaimed Amount in mergeTVS Causes Asset Shortfall and Prevents Complete Withdrawals
  
  

## Summary

The calculation of merge fees on the full flow amount instead of the unclaimed amount in `mergeTVS` will cause an asset shortfall in the contract for users and the protocol, as fees are taken from amounts that have already been partially claimed and transferred out, leaving insufficient tokens to fulfill all pending withdrawals.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:1040`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1037-L1040), the fee is calculated on `TVSToMerge.amounts[j]`, which represents the full flow amount. However, when users claim tokens via `claimTokens()` (line 974), those tokens are transferred out of the contract. The `amounts[j]` field still reflects the original full allocation, not the remaining unclaimed balance. As a result, fees are computed on amounts that may have already been partially claimed, creating a mismatch between the contract's actual token balance and the amounts tracked for fees and future withdrawals.

&nbsp;

## Internal Pre-conditions

- A user must have claimed at least a portion of tokens from one or more flows in a TVS before merging
- The `mergeFeeRate` must be set to a value greater than zero
- The contract must have sufficient tokens to cover fees calculated on the full flow amounts, but not enough to cover both those fees and all remaining unclaimed amounts

&nbsp;

## External Pre-conditions

- None required

&nbsp;

## Attack Path

1. User receives a TVS allocation with multiple flows, each with a total amount stored in `allocation.amounts[]`
2. User calls `claimTokens()` to claim a portion of vested tokens, which are transferred out of the contract (line 974)
3. The `allocation.amounts[]` values remain unchanged and still reflect the original full allocation amounts
4. User calls `mergeTVS()` to merge this partially-claimed TVS with another TVS
5. In `_merge()` at line 1040, the fee is calculated as `calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j])`, using the full flow amount instead of the unclaimed amount
6. The fee is deducted from the full amount and added to the merged TVS (line 1041), but the contract's actual token balance is lower than expected because part of the tokens were already transferred out
7. The contract attempts to transfer the calculated fee amount to the treasury (line 1024), but the raw balance may be insufficient to cover all fees and remaining unclaimed amounts
8. This creates a shortfall where the contract's token balance is less than the sum of all unclaimed amounts across all TVSs, preventing some users from completing their withdrawals

&nbsp;

## Impact

Users and the protocol suffer a loss proportional to the merge fee rate applied to already-claimed portions of flows. The contract's raw token balance becomes insufficient to process all pending withdrawals, causing some withdrawal transactions to revert due to insufficient balance. The severity increases with the number of merges performed on partially-claimed TVSs and the merge fee rate.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Calculate merge fees on the unclaimed amount rather than the full flow amount. For each flow being merged, compute the unclaimed amount using the formula: `unclaimedAmount = (amount * (vestingPeriod - claimedSeconds)) / vestingPeriod`, then calculate the fee on this unclaimed amount. Update line 1040 in `_merge()` to:

```js
uint256 vestingPeriod = TVSToMerge.vestingPeriods[j];
uint256 claimedSeconds = TVSToMerge.claimedSeconds[j];
uint256 unclaimedAmount = (TVSToMerge.amounts[j] * (vestingPeriod - claimedSeconds)) / vestingPeriod;
uint256 fee = calculateFeeAmount(mergeFeeRate, unclaimedAmount);
mergedTVS.amounts.push(unclaimedAmount - fee);
```

This ensures fees are only charged on tokens that are actually available in the contract and prevents the asset shortfall.
  