# [000476] A26ZDividendDistributor. Incorrect TVS Token Filter in getUnclaimedAmounts() Excludes Intended TVS Holders From Dividends
  
  ### Summary

Incorrect Token Filtering Logic Causes NFTs to Be Wrongly Excluded From Dividend Calculations in A26ZDividendDistributor::getUnclaimedAmounts

### Root Cause

The function getUnclaimedAmounts() compares:
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

According to the whitepaper and code structure, this contract is designed to distribute dividends to holders of TVSs for a specific token (e.g., A26Z). In that model:
* token in A26ZDividendDistributor is the TVS underlying token that should receive dividends.
* allocation.token in AlignerzVesting is also the TVS underlying token.

Therefore, the function should include NFTs where allocation.token == token and skip those with a different token. The current equality check does the opposite and returns 0 precisely when the tokens match, meaning the correct TVSs are ignored in the dividend calculation.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L126-L136

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls setUpTheDividends() to distribute rewards for TVSs of token X.
2. Function calls getTotalUnclaimedAmounts() → getUnclaimedAmounts(nftId).
3. getUnclaimedAmounts() checks
```solidity
if (token == allocation.token) return 0;
```
and skips all TVSs of token X (the ones meant to receive dividends).

4. Dividend calculation proceeds using zero valid TVSs.
5. Final result: legitimate TVS holders receive no dividends.

### Impact

Global dividend pool mis-accounting

getTotalUnclaimedAmounts() aggregates getUnclaimedAmounts() across all NFTs.

Because the core filter is inverted, totalUnclaimedAmounts does not represent the total unvested amount of the intended TVS token.

This corrupts all subsequent calculations that depend on this value.


### PoC

TODO

### Mitigation

_No response_
  