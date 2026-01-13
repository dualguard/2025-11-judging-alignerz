# [000010] Inverted check will block dividend distribution for all TVS holders
  
  ### Summary

An inverted token equality check in [`A26ZDividendDistributor::getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) will cause a complete block of dividend distribution for all TVS holders as the protocol will always treat valid TVS allocations as ineligible, compute a zero `totalUnclaimedAmounts`, and then revert with a division-by-zero when setting up dividends.

### Root Cause

In `A26ZDividendDistributor::getUnclaimedAmounts` the contract uses an inverted equality check to filter allocations:

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) {
    return 0;
}
```

This logic returns `0` and skips updating `unclaimedAmountsIn[nftId]` exactly when the allocation’s `token` matches the distributor’s configured `token`, so all relevant allocations are ignored while any mismatching allocations (if they ever exist) would be used instead. As a result, `getTotalUnclaimedAmounts()` sums zero for all valid TVS NFTs and `_setAmounts()` sets `totalUnclaimedAmounts` to `0`, which later causes `_setDividends()` to divide by zero when computing each holder’s `dividendsOf[owner].amount`.

### Internal Pre-conditions

1. The `vesting` contract is configured so that `vesting.allocationOf(nftId).token` equals the TVS `token` used by `A26ZDividendDistributor` for all relevant NFT IDs.
2. The distributor holds a positive stablecoin balance intended for distribution to TVS holders.
3. The owner calls `setUpTheDividends()` or calls `setAmounts()` followed by `setDividends()` to initialize and distribute dividends.

### External Pre-conditions

none.

### Attack Path

1. The protocol deploys and configures `A26ZDividendDistributor` and the associated `vesting` project so that each TVS NFT has an allocation whose `token` matches the distributor’s `token`.
2. The owner sends stablecoins to `A26ZDividendDistributor` and calls `setUpTheDividends()` (or `setAmounts()` then `setDividends()`).
3. Inside `_setAmounts()`, `getTotalUnclaimedAmounts()` iterates TVS NFTs and calls `getUnclaimedAmounts(nftId)`; for each owned NFT the inverted check sees matching tokens and immediately returns `0`, so `_totalUnclaimedAmounts` remains `0`.
4. `_setAmounts()` stores `totalUnclaimedAmounts = 0` and returns; then `_setDividends()` iterates over NFTs and computes each holder’s share as `unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts`, which attempts to divide by `0` and reverts.
5. The dividend setup transaction fails, `dividendsOf` is never populated, and the stablecoin balance remains stuck in the distributor until the owner manually withdraws it.

### Impact

The TVS NFT holders cannot receive any dividends from `A26ZDividendDistributor`, and up to the full `stablecoinAmountToDistribute` (i.e., the entire stablecoin balance held by the distributor) remains undistributed and effectively locked for TVS holders; only the owner can later recover these funds via `withdrawStuckTokens`, so users lose access to the dividends they should have received.

### PoC

_No response_

### Mitigation

Invert the token check so that only allocations whose `token` does not match the distributor’s configured `token` are ignored, and ensure the function proceeds to compute unclaimed amounts for the intended TVS token:

```solidity
if (address(token) != address(vesting.allocationOf(nftId).token)) {
    return 0;
}
// compute unclaimed amounts for the matching TVS token
```
  