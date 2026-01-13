# [000808] Dividends can be claimed in wrong token after stablecoin change
  
  ### Summary

`claimDividends()` transfers the user’s claimable dividends using the current stablecoin:
```solidity
stablecoin.safeTransfer(user, claimableAmount);
```
However, the owner can change the stablecoin to a different token between different distributions.

If a user has unclaimed dividends from a previous vesting period, their claim is executed in the new stablecoin, rather than the stablecoin originally intended.

### Root Cause

`claimDividends()` always transfers from the current stablecoin, ignoring which token the dividend was originally intended to be distributed in.

### Internal Pre-conditions

Owner must change the stablecoin.

### External Pre-conditions

None

### Attack Path

1. Alice has 100 tokens of dividend allocated in USDC.
2. The vesting period ends and Alice has not claimed her allocation yet.
3. The owner changes stablecoin to USDT for the next distribution period.
4. Alice calls `claimDividends()`.
5. Alice receives her unclaimed dividends in USDT instead of USDC.

### Impact

* The total `dividendsOf[user].amount` across all users will exceed the contract’s current stablecoin balance.
* Some users will not be able to fully claim their dividends.

### PoC

_No response_

### Mitigation

_No response_
  