# [000661] HIGH   Inverted token equality check causes valid TVS holders to always return zero unclaimed amounts, leading to division-by-zero revert in dividend setup
  
  ### Summary

An inverted token equality check in `A26ZDividendDistributor.getUnclaimedAmounts` will cause a division-by-zero revert for the owner as the owner will be unable to set up dividends because valid TVS NFTs with matching tokens always return `0` unclaimed amounts, resulting in `totalUnclaimedAmounts = 0` which is used as a denominator in `_setDividends`.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141C8-L141C84


In `A26ZDividendDistributor.sol` at `getUnclaimedAmounts(uint256 nftId)`, the token check `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;` is logically inverted: it returns `0` when the distributor's token **matches** the TVS allocation token, when the intended behavior is to return `0` for **mismatched** tokens and compute unclaimed amounts for matching ones. 

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; // BUG: should be !=
    // ... rest of computation
}
```

This causes correctly configured NFTs (where the distributor is set up for the right token) to be excluded from dividend calculations, while only misconfigured NFTs would enter the computation path.



### Internal Pre-conditions

1. Owner needs to deploy `A26ZDividendDistributor` with the `_token` parameter set to the same ERC20 address as the vesting project's TVS token (the correct configuration).

2. At least one NFT needs to be minted via the vesting contract, so `vesting.allocationOf(nftId).token` returns the project's token address and `amounts.length > 0`.

3. Owner needs to call `setUpTheDividends()` or `setAmounts()` to initialize dividend distribution.


### External Pre-conditions

none

### Attack Path

1. Owner deploys `A26ZDividendDistributor` with the correct `_token` address matching the vesting project's TVS token.

2. Bidders participate in the vesting project, bids are finalized, and at least one NFT is claimed, so `vesting.allocationOf(nftId)` contains valid allocation data with `amounts[i] > 0`.

3. Owner deposits stablecoin rewards into the distributor contract to prepare for dividend distribution.

4. Owner calls `setUpTheDividends()`, which internally calls `_setAmounts()`.

5. `_setAmounts()` calls `getTotalUnclaimedAmounts()`, which iterates over all minted NFTs and calls `getUnclaimedAmounts(i)` for each owned NFT.

6. For each valid NFT where `address(token) == address(vesting.allocationOf(i).token)` (the correct case), `getUnclaimedAmounts(i)` hits the early return and returns `0` instead of computing the actual unclaimed token value.
7. As a result, `totalUnclaimedAmounts` is set to `0` even though TVS holders have real unclaimed token allocations.

8. When `_setDividends()` is called (either immediately after in `setUpTheDividends()` or later via `setDividends()`), it attempts to compute `dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)`.

9. The transaction reverts with a division-by-zero error because `totalUnclaimedAmounts == 0`, permanently preventing dividend setup and distribution.



### Impact

The protocol cannot distribute dividends to TVS holders, as the owner is unable to complete the dividend setup process due to a guaranteed division-by-zero revert in `_setDividends`. Stablecoin rewards deposited into the distributor contract become locked and unusable for their intended purpose until the contract logic is fixed or the owner withdraws them via `withdrawStuckTokens`. TVS holders lose access to their rightful dividend rewards.

### PoC

_No response_

### Mitigation

_No response_
  