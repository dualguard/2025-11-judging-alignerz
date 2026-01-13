# [000681] [M-1] Dividend setup can revert/lock funds when total unclaimed is zero
  
  ### Summary

The dividend distributor divides by `totalUnclaimedAmounts` without guarding against zero; if unclaimed amounts are zero (or stale and computed as zero), `_setDividends` reverts, blocking dividend setup and trapping deposited stablecoins.

### Root Cause

In `A26ZDividendDistributor.sol:206-224`, `_setAmounts` sets `totalUnclaimedAmounts = getTotalUnclaimedAmounts()`, and `_setDividends` then computes `unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts` without checking `totalUnclaimedAmounts > 0`.

### Internal Pre-conditions

  1. Owner calls `setUpTheDividends`/`setDividends`/`setAmounts`.
  2. `getTotalUnclaimedAmounts()` returns 0 (e.g., all NFTs fully claimed, or miscomputed/stale).

### External Pre-conditions

None.

### Attack Path

  1. Owner deposits stablecoin into the distributor and calls `setUpTheDividends`.
  2. `getTotalUnclaimedAmounts()` returns 0.
  3. `_setDividends` divides by `totalUnclaimedAmounts` and reverts; stablecoin remains stuck, dividends never configured.

### Impact

Dividend configuration halts; deposited stablecoin cannot be distributed. Users expecting dividends receive nothing until a redeploy or code fix.

### PoC

_No response_

### Mitigation

Guard for zero: if `totalUnclaimedAmounts == 0`, either revert with a clear error before division or skip distribution and allow withdrawal; ensure `unclaimedAmountsIn` is refreshed just before computing totals to avoid stale zeros.
  