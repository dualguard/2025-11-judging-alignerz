# [000809] Total user dividends will exceed available funds
  
  ### Summary

The `setAmounts()` flow calculates the total amount to distribute for a new dividend period as:
```solidity
stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
```
If a user has unclaimed dividends from the prior vesting period, that amount is not reset, but `stablecoinAmountToDistribute` includes this leftover amount as it is simply the balance of the contract.

This causes `dividendsOf[owner].amount` across all users to exceed the actual contract balance.

### Root Cause

`_setAmounts()` uses contract balance to determine the total distributable amount.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

* Users can receive more dividends than intended.
* The total `dividendsOf[user].amount` across all users can exceed the contract’s stablecoin balance.
* Some users may not be able to fully claim their dividends.

### PoC

_No response_

### Mitigation

_No response_
  