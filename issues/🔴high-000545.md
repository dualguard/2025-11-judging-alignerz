# [000545] Incorrect Fee Calculation on Full Flow Amount in splitTVS Causes Asset Shortfall and Prevents Withdrawals
  
  
## Summary

The `splitTVS` function calculates fees on the full original flow amounts instead of the unclaimed amounts, causing an asset shortfall in the contract. This occurs because fees are computed on tokens that may have already been claimed and sent to users, leaving insufficient balance to process the fee transfer and potentially blocking future withdrawals.

## Root Cause

In [`AlignerzVesting.sol:1068-1070`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1053-L1071), the `splitTVS` function retrieves `allocation.amounts` (the full original allocation amounts) and passes them to `calculateFeeAndNewAmountForOneTVS` to compute fees. The function should use the unclaimed amounts (original amount minus claimed amount) per flow, but it uses the full original amounts. Since users can claim tokens before splitting, the contract may attempt to transfer fees based on tokens that are no longer in the contract, creating a shortfall.

## Internal Pre-conditions

1. A user must have a TVS allocation with at least one flow that has been partially or fully claimed
2. The `splitFeeRate` must be set to a value greater than 0
3. The user must call `splitTVS` after claiming some tokens from their TVS allocation
4. The contract's token balance must be less than the sum of all unclaimed amounts plus the fees calculated on the full flow amounts

## External Pre-conditions

None required. This is an internal logic issue that manifests when users claim tokens before splitting.

## Attack Path

1. User receives a TVS allocation with multiple flows (e.g., 1000 tokens per flow)
2. User calls `claimTokens` to claim a portion of their vested tokens (e.g., 500 tokens), reducing the contract's balance
3. User calls `splitTVS` with the same TVS
4. `splitTVS` retrieves `allocation.amounts` containing the full original amounts (1000 tokens) at line 1068
5. `splitTVS` calls `calculateFeeAndNewAmountForOneTVS` with the full amounts at line 1070, calculating fees on 1000 tokens instead of the remaining 500 unclaimed tokens
6. The contract attempts to transfer the fee amount calculated on the full flow amount at line 1072, but the contract balance is insufficient because part of the tokens were already sent to the user in step 2
7. The transaction reverts due to insufficient balance, or if it succeeds in some edge cases, it creates an accounting mismatch where the contract's raw balance is less than the sum of all unclaimed allocations

## Impact

Users suffer a loss proportional to the fees charged on claimed flows amount. The loss particularly affects the last users to withdraw from the contract, as the contract's token balance becomes insufficient to fulfill all withdrawal requests. 

## Proof of Concept
N/A

## Mitigation

Modify the `splitTVS` function to calculate fees based on unclaimed amounts rather than full flow amounts. For each flow, compute the unclaimed amount by subtracting the already claimed portion from the original amount, then calculate fees on that unclaimed amount:

```js
// Calculate unclaimed amounts for each flow
uint256[] memory unclaimedAmounts = new uint256[](nbOfFlows);
for (uint256 i; i < nbOfFlows; i++) {
    uint256 originalAmount = allocation.amounts[i];
    uint256 claimedAmount = (originalAmount * allocation.claimedSeconds[i]) / allocation.vestingPeriods[i];
    unclaimedAmounts[i] = originalAmount - claimedAmount;
}

// Calculate fees on unclaimed amounts instead of full amounts
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, unclaimedAmounts, nbOfFlows);
```

This ensures fees are only charged on tokens that are still available in the contract and prevents the asset shortfall.
  