# [000588] Lack of Token-Specific Accounting in Dividend Distribution Causes Incorrect Reward Calculations and Potential Claim Failures When Stablecoin is Changed
  
  

&nbsp;

## Summary

The lack of separate balance tracking per stablecoin token in `A26ZDividendDistributor.sol` will cause incorrect reward accounting and potential claim failures for TVS holders as the owner can change the stablecoin and add dividends denominated in different tokens with different decimal precisions to the same storage variable.
Note that there's no claiming window or deadline in A26Z contract, so it's normal for a user to not claim their remaining dividends for a long time.

&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:218`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218), the `_setDividends()` function uses `+=` to add new dividend amounts to existing `dividendsOf[owner].amount` values without tracking which stablecoin token each amount represents. The choice to use a single `uint256` value per user without token context is a mistake, as different stablecoins have different decimal precisions (e.g., USDC/USDT use 6 decimals while DAI uses 18 decimals), causing values of incompatible units to be added together when the stablecoin is changed via `setStablecoin()` at line 84-87.
Other than the difference in token units and decimals, the raw balance of the different tokens and the contract state accounting will be mismatched for all the users, cause insufficient balance during withdrawals.

```js
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

&nbsp;

## Internal Pre-conditions

1. Owner needs to call `setStablecoin()` to change the `stablecoin` variable from one token to another token with different decimal precision
2. Owner needs to call `setAmounts()` or `setUpTheDividends()` to set `stablecoinAmountToDistribute` using the new stablecoin's balance
3. Owner needs to call `setDividends()` or `setUpTheDividends()` to execute `_setDividends()` which adds new dividend amounts to existing `dividendsOf[owner].amount` values
4. At least one user must have existing `dividendsOf[owner].amount > 0` from a previous dividend distribution using a different stablecoin

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. Owner calls `setStablecoin(newStablecoin)` to change the stablecoin from Token A (e.g., USDC with 6 decimals) to Token B (e.g., DAI with 18 decimals)
2. Owner deposits Token B into the contract
3. Owner calls `setUpTheDividends()` which calls `_setAmounts()` to set `stablecoinAmountToDistribute` to the balance of Token B
4. Owner calls `setDividends()` or `setUpTheDividends()` which calls `_setDividends()` to distribute dividends
5. `_setDividends()` adds new dividend amounts (in Token B units) to existing `dividendsOf[owner].amount` values (in Token A units) using `+=` operator
6. User attempts to claim dividends via `claimDividends()`
7. `claimDividends()` calculates `claimableAmount` from the incorrectly mixed `dividendsOf[user].amount` and attempts to transfer using the current `stablecoin` (Token B)

&nbsp;

## Impact

TVS holders suffer incorrect reward accounting, leading to overpayment or underpayment of rewards, and possible insufficient balance withdrawal failures. 

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Track dividends separately per stablecoin token by using a nested mapping: `mapping(address => mapping(IERC20 => Dividend)) dividendsOf` to store dividend amounts per user per token.

  