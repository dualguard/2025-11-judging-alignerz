# [000689] Logic inversion in `getUnclaimedAmounts` will cause zero dividend allocation for all legitimate TVS holders
  
  ### Summary

An incorrect equality check in `getUnclaimedAmounts` will cause a complete loss of dividend rewards for TVS holders, as the system explicitly filters out the correct token from the calculation logic, resulting in a calculated share of zero.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140
In `A26ZDividendDistributor.sol`, within the `getUnclaimedAmounts` function, there is a logic inversion in the token validation check:

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

The `token` variable represents the specific project token being tracked for dividend calculations. The intention is presumably to exclude NFTs that hold a *different* token (e.g., from a different project). However, the code uses `==` instead of `!=`.

This means:
- If the NFT holds the **correct** token, the condition is `true`, and the function returns `0`.
- If the NFT holds an **incorrect** token, the code proceeds .

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.  The Owner calls `setUpTheDividends()` to calculate and distribute rewards.
2.  The function calls `_setAmounts()` -> `getTotalUnclaimedAmounts()` -> `getUnclaimedAmounts(nftId)`.
3.  Inside `getUnclaimedAmounts`, the contract checks if the distributor's target `token` matches the TVS's `token`.
4.  Since they match (as intended for a valid claim), the condition `address(token) == address(vesting.allocationOf(nftId).token)` evaluates to `true`.
5.  The function returns `0` immediately.
6.  The `unclaimedAmountsIn[nftId]` is set to 0, and subsequently, `dividendsOf[owner].amount` remains 0.

### Impact

The TVS holders suffer a 100% loss of their eligible dividends. The dividend distribution mechanism fails to allocate any funds to legitimate holders of the project token.

### PoC

_No response_

### Mitigation

Reverse the condition to check for inequality. The function should return 0 only if the tokens do **not** match.

```diff
- if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+ if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  