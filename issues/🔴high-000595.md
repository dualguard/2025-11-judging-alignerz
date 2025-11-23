# [000595] Inverted Logic Bug in `getUnclaimedAmounts` Function Causes Incorrect Early Return and Excludes All Matching TVS Allocations from Dividend Distribution
  
  

## Summary

An inverted comparison operator in `getUnclaimedAmounts` causes a complete loss of dividend eligibility for TVS holders as the function returns 0 when tokens match instead of when they don't match, incorrectly excluding all matching TVS allocations from dividend calculations.
```js
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
		// ...
	}
```
## Root Cause

In [`A26ZDividendDistributor.sol:141`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141), the comparison operator is inverted. The check `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;` returns 0 when tokens match, but it should return 0 when tokens don't match. The function should calculate unclaimed amounts for NFTs matching the distributor's token and skip NFTs with different tokens. Using `==` instead of `!=` causes all matching TVS allocations to be excluded from dividend distribution.

## Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` or `setAmounts()` to trigger `getTotalUnclaimedAmounts()` which calls `getUnclaimedAmounts()` for each NFT
2. At least one NFT must exist with an allocation that matches the `token` address set in the distributor contract
3. The `token` variable must be set to a valid token address (either in constructor or via `setToken()`)

## External Pre-conditions

None required.

## Attack Path

1. Owner deploys `A26ZDividendDistributor` with a specific `_token` address to distribute dividends for that token's TVS holders
2. TVS allocations are created in the vesting contract with NFTs that have allocations matching the distributor's token
3. Owner calls `setUpTheDividends()` or `setAmounts()` to calculate total unclaimed amounts
4. `getTotalUnclaimedAmounts()` iterates through all NFTs and calls `getUnclaimedAmounts()` for each owned NFT
5. For each NFT with a matching token, `getUnclaimedAmounts()` hits line 141 and incorrectly returns 0 due to the inverted logic
6. `totalUnclaimedAmounts` is calculated with missing values (all matching allocations return 0)
7. Owner calls `setDividends()` to distribute dividends based on the incorrect `totalUnclaimedAmounts`
8. TVS holders with matching tokens receive no dividends or incorrect dividend amounts

## Impact

TVS holders with allocations matching the distributor's token suffer a complete loss of dividend eligibility. Their unclaimed amounts are incorrectly calculated as 0, so they are excluded from dividend distribution.

## Proof of Concept
N/A

## Mitigation

Change line 141 to:
```diff
+ if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  