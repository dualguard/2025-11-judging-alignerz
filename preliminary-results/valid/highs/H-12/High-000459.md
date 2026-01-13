# [000459] Incorrect check of allocation token for dividends will prevent users from earning their allocation
  
  ### Summary

In the distributor contract, the function `getUnclaimedAmounts` has a condition to calculate only the amounts of allocations that have the same tokens as the distributor contract. However, the check returns 0 if the tokens are the same, while it should return 0 otherwise.

### Root Cause

`getUnclaimedAmounts` returns 0 if `A26ZDividendDistributor.token == vesting.allocationOf(nftId).token` (it should be the opposite).

### Attack Path

1. Alice has an allocation for USDC
2. The dividend distributor is set to distribute USDC
3. `getUnclaimedAmounts` returns 0 for Alice's NFT

### Impact

Users can't claim their rightful dividends.

### Mitigation

Update the condition to return 0 if the allocation token is not correct.

```diff
-   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+   if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  