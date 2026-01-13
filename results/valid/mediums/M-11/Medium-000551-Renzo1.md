# [000551] Unclaimed Dividends Redistributed to All Users When `_setAmounts` is Called Again Due to Using Raw Contract Balance Instead of Tracking New Deposits
  
  

## Summary

Using the raw contract balance to set `stablecoinAmountToDistribute` in `_setAmounts` causes unclaimed dividends to be included in new distributions. When the owner calls `setUpTheDividends()` again, unclaimed stablecoins are redistributed proportionally to all users instead of being reserved for the original recipients, causing a loss for users who haven't claimed yet.

```js
    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }
```
## Root Cause

In [`A26ZDividendDistributor.sol:207-211`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L206-L211), `_setAmounts()` sets `stablecoinAmountToDistribute = stablecoin.balanceOf(address(this))` at line 208. This includes both new deposits and unclaimed dividends still in the contract. When `_setDividends()` runs at line 218, it distributes the entire balance proportionally, so unclaimed dividends are redistributed to all users rather than reserved for the original recipients.

## Internal Pre-conditions

1. The owner must have called `setUpTheDividends()` at least once to set up an initial dividend distribution
2. At least one user must not have claimed their dividends, leaving stablecoins in the contract
3. The owner must call `setUpTheDividends()` again (for a new distribution or top-up) while unclaimed dividends remain in the contract

## External Pre-conditions

None required. The vulnerability triggers based on the contract's internal state and owner actions.

## Attack Path

1. The owner calls `setUpTheDividends()` to set up the first dividend distribution, allocating stablecoins to users based on their unclaimed TVS amounts
2. Some users do not claim their dividends, leaving unclaimed stablecoins in the contract
3. The owner deposits additional stablecoins or calls `setUpTheDividends()` again for a new distribution or top-up
4. `_setAmounts()` executes and sets `stablecoinAmountToDistribute = stablecoin.balanceOf(address(this))`, which includes both new deposits and unclaimed dividends from step 2
5. `_setDividends()` executes and distributes the entire balance proportionally to all users based on their unclaimed amounts at line 218
6. The unclaimed dividends from step 2 are redistributed to all users instead of being reserved for the original recipients
7. Users who haven't claimed yet lose their dividends, as they are diluted across all users

## Impact

Users who haven't claimed their dividends suffer a loss proportional to the unclaimed amount that gets redistributed. The loss increases with more unclaimed dividends and more frequent calls to `setUpTheDividends()`. The loss falls on the last users to claim their dividends as users who claim early benefit from the redistribution, while late claimers lose their allocated dividends since the raw balance won't be up to the individual accounting.

## Proof of Concept
N/A

## Mitigation

Track new deposits separately from unclaimed dividends. Modify `_setAmounts()` to only include newly deposited funds.
  