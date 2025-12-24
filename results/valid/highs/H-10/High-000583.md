# [000583] Split Fee Calculation on Fully Claimed Flows Causes Double Spending and Insufficient Funds for Late Withdrawers
  
  
## Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` charges fees on the entire flow amount, including flows that have been fully claimed (`claimedFlows[i] == true`). This causes a double-spending issue where tokens are first transferred to users when claimed, and then fees are charged again on those same tokens during split operations. The loss falls on the last users to withdraw from the protocol as the contract's token balance becomes insufficient to fulfill all withdrawal requests.

## Root Cause

In [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the `splitTVS` passes the all the flow amounts in the TVS, including fully claimed flows, to `calculateFeeAndNewAmountForOneTVS` function which iterates through all flows and calculates fees on the full `amounts[i]` value without checking whether the flow has been fully claimed. When a flow is fully claimed via `claimTokens()` in `AlignerzVesting.sol:942-976`, the tokens are transferred to the user and `claimedFlows[i]` is set to `true`, but `allocation.amounts[i]` still contains the original full allocation amount. The function should only charge fees on the unclaimed portion of each flow, as tokens from fully claimed flows no longer exist in the contract.

## Internal Pre-conditions

1. At least one TVS flow must be fully claimed (`claimedFlows[i] == true`) where tokens have been transferred to the user via `claimTokens()`
2. The owner of a TVS with at least one fully claimed flow must call `splitTVS()` or `mergeTVS()` to trigger the fee calculation
3. The `splitFeeRate` or `mergeFeeRate` must be greater than 0 to charge fees

## External Pre-conditions

None required.

## Attack Path

1. User A claims tokens from their TVS, fully claiming one or more flows. Tokens are transferred to User A, and `claimedFlows[i]` is set to `true` for those flows, but `allocation.amounts[i]` still contains the original full amount.
2. User A (or any user with a TVS containing fully claimed flows) calls `splitTVS()` or `mergeTVS()`.
3. `splitTVS()` at line 1070 or `mergeTVS()` at line 1014 calls `calculateFeeAndNewAmountForOneTVS()` with all flow amounts, including fully claimed flows.
4. `calculateFeeAndNewAmountForOneTVS()` calculates fees on the full `amounts[i]` for all flows, including those where `claimedFlows[i] == true`.
5. The calculated fee amount is transferred to the treasury via `token.safeTransfer(treasury, feeAmount)` at line 1072 or 1024.
6. For fully claimed flows, the contract attempts to transfer tokens that no longer exist (they were already transferred to users in step 1), causing a mismatch between the contract's actual balance and expected balance.
7. Later users attempting to withdraw their tokens may find insufficient funds, as the contract's balance has been depleted by fees charged on already-claimed tokens.

## Impact

Users suffer a loss proportional to the fees charged on fully claimed flows. The loss particularly affects the last users to withdraw from the contract, as the contract's token balance becomes insufficient to fulfill all withdrawal requests. 


## Proof of Concept
N/A

## Mitigation

Modify `splitTVS` implementation to remove the amount for claimed flows before calling `calculateFeeAndNewAmountForOneTVS`.

  