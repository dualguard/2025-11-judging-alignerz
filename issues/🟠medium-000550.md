# [000550] Lack of TVS Position Maintenance Requirement in Dividend Distribution Allows TVS Holders to Maximize Dividends by Delaying Token Claims and Then Claiming All Tokens Without Maintaining Position
  
  

## Summary

The lack of a requirement to maintain a TVS position after dividends are set in `A26ZDividendDistributor.sol` will cause an unfair distribution of dividends for the protocol and other TVS holders as a TVS holder will delay claiming their tokens to maximize their unclaimed amount when `_setDividends()` is called, then claim all their tokens or transfer their NFT while still receiving the full dividend amount that was calculated based on their snapshot position.

```js
    function _setDividends() internal {
        // ...
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            // ...
    }
```

&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:214-224`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224), the `_setDividends()` function calculates and assigns dividend amounts to TVS holders based on their unclaimed token amounts at the time of the snapshot. Once `dividendsOf[owner].amount` is set in line 218, there is no mechanism to verify that the user maintains their TVS position. The `claimDividends()` function in lines 187-204 allows users to claim their vested dividends over time without checking whether they still own the NFT or have unclaimed tokens remaining. This design allows TVS holders to maximize their dividend allocation by delaying token claims before the snapshot, then claim all their tokens or transfer their NFT while continuing to receive dividends based on the historical snapshot.

&nbsp;

## Internal Pre-conditions

1. Owner needs to call `setDividends()` or `setUpTheDividends()` to execute `_setDividends()` and set dividend allocations
2. TVS holder needs to have an NFT with unclaimed tokens allocated to them
3. TVS holder needs to delay claiming their tokens via `claimTokens()` in the vesting contract before `_setDividends()` is called
4. `stablecoinAmountToDistribute` needs to be greater than zero when `_setDividends()` is called

&nbsp;

## External Pre-conditions

None required.

&nbsp;

## Attack Path

1. TVS holder receives an NFT with vested tokens but delays claiming any tokens via `claimTokens()` in the vesting contract
2. Owner calls `setDividends()` or `setUpTheDividends()`, which executes `_setDividends()` and calculates dividends based on current unclaimed amounts
3. `_setDividends()` calls `getUnclaimedAmounts()` for each NFT, which calculates the unclaimed token value based on the current vesting state
4. The TVS holder's address is set as the dividend benefactor with `dividendsOf[owner].amount` calculated proportionally based on their high unclaimed amount
5. TVS holder calls `claimTokens()` in the vesting contract to claim all their remaining tokens, or transfers/sells their NFT
6. TVS holder calls `claimDividends()` repeatedly over the vesting period to receive the full dividend amount that was calculated based on their snapshot position, even though they no longer maintain their TVS position

&nbsp;

## Impact

The protocol and other TVS holders suffer an unfair distribution of dividends. TVS holders who delay claiming tokens before the dividend snapshot receive disproportionately higher dividends, while those who claim tokens earlier receive lower dividends. After the snapshot, these holders can claim all their tokens or transfer their NFTs while still receiving dividends based on their historical unclaimed position, effectively receiving dividends without maintaining the required TVS position. The attacker gains additional dividends at the expense of other TVS holders who maintain their positions.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Consider implementing a mechanism to verify TVS position maintenance when claiming dividends.
  