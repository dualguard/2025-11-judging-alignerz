# [000217] Incorrect token-matching condition bypasses dividend allocation for all main-token TVSs
  
  ### Summary

[`A26ZDividendDistributor.getUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) contains the condition:
```js
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

This causes all TVSs vesting the main project token (the only TVSs intended to receive dividends) to return 0 unclaimed amount, resulting in zero dividends assigned during [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218C17-L218C24).

As a result, no main-token TVS receives dividends at all, despite being the only category eligible according to the team.

> dividends are intended to apply to TVSs vesting main project token

### Root Cause

Incorrect token comparison logic:

The contract stores:

```js
IERC20 token; 
IERC20 stablecoin;        
```

Dividends should apply only to TVSs vesting the main project token (token).

Instead, the code excludes them:

```js
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
→ If vesting token equals the configured main token, return 0. See Attack Path for details.

### Internal Pre-conditions

1. The dividend distributor is initialized with:
   - token (the main project token)
   - stablecoin (token used for dividend payouts)
2. TVSs exist that vest token.
3. Admin calls either:
   - setUpTheDividends(),
   - setAmounts() then setDividends().

### External Pre-conditions

None.

### Attack Path

Consider the following example:

1. Admin funds the dividend contract with `stablecoin`.
2. Admin calls:

```js
_setAmounts();   // collects unclaimed amounts for each NFT
_setDividends(); // assigns dividends based on unclaimed proportions
```

3. In `_setAmounts()`, `getUnclaimedAmounts(nftId)` is called.
4. For any NFT vesting the main token:

```js
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0
```
→ unclaimedAmountsIn[nftId] = 0

5. Then `_setDividends()` allocates:

```js
dividendsOf[owner].amount += (0 * stablecoinAmountToDistribute / totalUnclaimedAmounts)
```
→ All main-token TVSs receive zero dividends.

Dividends are instead allocated only to TVSs that vest other tokens — contradicting the intended design.

### Impact

Due to a incorrect token-matching condition, all TVSs vesting the main project token — the only ones intended to receive dividends — are incorrectly excluded from dividend calculations. As a result, no eligible TVS receives any dividends, breaking dividend distribution.

### PoC

_No response_

### Mitigation

Replace the incorrect condition:

`if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;`

with 

`if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;`


This ensures that only TVSs vesting the main project token qualify for dividends.
  