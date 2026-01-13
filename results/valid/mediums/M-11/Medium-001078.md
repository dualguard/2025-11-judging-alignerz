# [001078] Repeated Calls to setUpTheDividends() Inflate User Rewards
  
  ### Summary

Calling `setUpTheDividends() `multiple times causes `_setAmounts()` and `_setDividends()` to re-execute fully. Because dividend amounts are added using +=, users receive dividends multiple times for the same unclaimed amounts. This drains the distributor’s stablecoin balance and results in early callers receiving inflated dividends, leaving insufficient funds for later users.

### Root Cause

In A26ZDividendDistributor.sol:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111

`_setAmounts()` recomputes `unclaimedAmountsIn[nftId]` for all NFTs every time it is called.

`_setDividends()` redistributes the entire stablecoinAmountToDistribute again using:

```solidity
dividendsOf[owner].amount += (
    unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
);
```

Because += is used instead of assigning once, calling setUpTheDividends() repeatedly multiplies the same reward.

There is **no state guard preventing** multiple initializations.

### Internal Pre-conditions

1. Owner (admin) must call `setUpTheDividends()` more than once.


### External Pre-conditions

_No response_

### Attack Path

1. Owner mistakenly calls `setUpTheDividends()` again for the same project.
2. `_setAmounts()` recalculates unclaimed amounts.
3. `_setDividends()` adds new dividend amounts on top of previous ones.
4. Early claimers withdraw inflated rewards.
5. Contract balance becomes insufficient for remaining users -> later users cannot claim.

### Impact

The protocol’s stablecoin pool becomes insufficient to pay all users because:
- Users who claim early gain multiple times their rightful dividend.
- Late claimers may receive zero because funds get drained.

### PoC

_No response_

### Mitigation

Add a guard so dividends can only be initialized once per project:
  