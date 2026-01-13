# [000557] Integer division precision loss in `getUnclaimedAmounts()` calculation causes artificial inflation of unclaimed amounts leading to incorrect dividend distribution
  
  

## Summary

Integer division precision loss in the `claimedAmount` calculation will cause incorrect dividend distribution for the protocol and other TVS holders as an attacker can split TVS into small amounts where `claimedAmount` rounds down to zero, artificially inflating their `unclaimedAmountsIn` value and receiving higher dividends than deserved.
```js
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // ...
        for (uint i; i < len;) {
            // ...
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            // ...
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

## Root Cause

In [`A26ZDividendDistributor.sol:153`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L153), the calculation `claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i]` uses integer division. When `claimedSeconds[i] * amounts[i]` is less than `vestingPeriods[i]`, the result rounds down to zero due to Solidity's integer division truncation. This causes `unclaimedAmount = amounts[i] - claimedAmount` to equal the full `amounts[i]` instead of the correct partially-claimed amount, artificially inflating the `unclaimedAmountsIn` value used for dividend calculations.

The design allows TVS splitting (via `AlignerzVesting::splitTVS`) which can create very small `amounts[i]` values through percentage-based splitting, making this precision loss more likely to occur.

## Internal Pre-conditions

1. A TVS must be split into smaller amounts such that at least one flow has `amounts[i]` small enough that `claimedSeconds[i] * amounts[i] < vestingPeriods[i]`
2. The vesting must have progressed such that `claimedSeconds[i] > 0` but `claimedSeconds[i] * amounts[i] < vestingPeriods[i]`
3. The owner must call `setUpTheDividends()` or `setDividends()` after the precision loss has occurred to calculate dividends based on the inflated `unclaimedAmountsIn` values

## External Pre-conditions

None required. The vulnerability is inherent to the integer division operation and does not depend on external factors.

## Attack Path

1. Attacker owns a TVS with vesting flows containing `amounts[i]` values
2. Attacker calls `splitTVS()` to split the TVS into multiple smaller TVSs, creating flows with very small `amounts[i]` values (e.g., through small percentage splits)
3. Time passes and vesting progresses, so `claimedSeconds[i] > 0` for some flows
4. For flows where `claimedSeconds[i] * amounts[i] < vestingPeriods[i]`, when `getUnclaimedAmounts()` is called (either by attacker or during `setUpTheDividends()`), the calculation at line 153 results in `claimedAmount = 0` due to integer division rounding down
5. The `unclaimedAmount` calculation at line 154 becomes `amounts[i] - 0 = amounts[i]`, incorrectly treating the full amount as unclaimed even though some portion should be considered claimed
6. This inflated `unclaimedAmountsIn[nftId]` value is used in dividend distribution calculations (line 218), causing the attacker to receive higher dividends than they should based on their actual unclaimed amounts
7. Other TVS holders receive proportionally less dividends due to the incorrect total unclaimed amounts calculation

## Impact

The protocol and other TVS holders suffer incorrect dividend distribution. Attackers who split TVS into small amounts can receive higher dividends than deserved when precision loss occurs. The severity depends on how many small-amount flows exist and how much the precision loss inflates the unclaimed amounts. In extreme cases where many flows round down to zero, the attacker could receive dividends based on nearly the full `amounts[i]` value even when a significant portion should be considered claimed.

## Proof of Concept
N/A


## Mitigation

  