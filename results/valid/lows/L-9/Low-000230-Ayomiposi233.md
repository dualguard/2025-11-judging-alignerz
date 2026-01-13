# [000230] Precision Loss in `claimTokens` Leads to Permanently Stuck Dust Tokens
  
  ### Summary

The `claimTokens` function, which allows users to withdraw their vested tokens, contains a logic flaw that makes it impossible to claim the full vested amount. The function relies on a helper, `getClaimableAmountAndSeconds`, which calculates the claimable amount using integer division. This helper function includes a strict check that reverts the transaction if the calculated amount rounds down to zero. Consequently, when a user tries to claim the final small fraction of their vesting period, the calculation results in zero, causing the claim to fail. This prevents users from ever withdrawing the "dust" remainder of their tokens, resulting in a permanent loss of funds for every vesting flow.

### Root Cause

The root cause of this vulnerability lies in the `getClaimableAmountAndSeconds` function.

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996

1. The function calculates the claimable amount using integer division: `claimableAmount = (amount * claimableSeconds) / vestingPeriod;`.
2. Due to the nature of integer division in Solidity, if the product `(amount * claimableSeconds)` is less than vestingPeriod, the result `claimableAmount` will be 0.
3. The function then enforces a strict check: `require(claimableAmount > 0, No_Claimable_Tokens());`

### Internal Pre-conditions

- A user owns a vesting NFT (`nftId`) with at least one active vesting flow.
- The vesting flow is partially but not fully claimed.
- The remaining claimable time (`claimableSeconds`) is small, such that `(total_flow_amount * remaining_claimable_seconds) < vesting_period`.

### External Pre-conditions

- The user calls the `claimTokens(projectId, nftId)` function to withdraw their vested tokens.

### Attack Path

This is not an attack path but rather a standard user flow that demonstrates the bug.

1. __Setup__:

- A user has a vesting flow with `amount = 1000` tokens and `vestingPeriod = 2,592,000` seconds (`30 days`).
- The user has already claimed tokens corresponding to `2,591,999` seconds.
- The remaining claimableSeconds is `1`.
2. __Action__:

- The user calls `claimTokens` to claim their final remaining tokens.
3. __Execution Flow__:

- The `claimTokens` function calls `getClaimableAmountAndSeconds`.
- Inside the helper, `claimableAmount` is calculated: `(1000 * 1) / 2592000`.
- Because `1000 < 2592000`, the integer division results in `claimableAmount = 0`.
- The check `require(claimableAmount > 0, ...)` is executed. Since `claimableAmount` is `0`, the condition is false.
4. __Outcome (Permanent Fund Loss)__:

- The transaction reverts with the `No_Claimable_Tokens` error.
- The user can never successfully call `claimTokens` again for this flow because the claimable amount will always be calculated as zero. The final dust tokens corresponding to that last second of vesting are permanently and irrecoverably stuck in the contract.

### Impact

The impact is a permanent loss of funds for users. Although the amount lost per flow may be small ("dust"), this happens for every single vesting flow across all users of the protocol. This leads to an accumulation of locked funds in the contract that belong to users but can never be retrieved. It breaks the core promise of a vesting contract, which is to allow users to eventually claim 100% of their entitled tokens.

### PoC

_No response_

### Mitigation

The mitigation involves removing the strict `require` check from the helper function and instead handling the zero-claim case gracefully within the `claimTokens` function. This ensures that transactions don't revert for dust amounts and allows the rest of the claiming logic to proceed.
  