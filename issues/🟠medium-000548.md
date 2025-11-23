# [000548] Overly Broad Exception Handling in `safeOwnerOf` Masks Critical System Failures and Causes Users to Miss Dividend Allocations.
  
  

## Summary

The catch-all exception handler in `safeOwnerOf` that treats all reverts from `extOwnerOf` as non-existent tokens will cause TVS holders to miss dividend allocations as the owner calls `setDividends()` which iterates over NFTs, and when `extOwnerOf` reverts due to gas exhaustion on later iterations, these failures are silently treated as non-existent tokens, causing those NFTs to be skipped in dividend calculations while the transaction completes successfully due to the 63/64 gas rule.
```js
    function safeOwnerOf(...) public view returns (address owner, bool exists) {
        try nft.extOwnerOf(nftId) returns (address _owner) {
            // ...
        } catch {
            // call reverted => token does not exist (burned or never minted)
			// @audit this can also revert due to gas exhaustion
            return (address(0), false);
        }
    }
```

Some other reasons the Txn could revert are
- Contract being paused
- NFT contract being upgraded/changed

&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:165-173`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L165-L173), the `safeOwnerOf()` function uses a catch-all `catch {}` block that treats any revert from `nft.extOwnerOf(nftId)` as if the token does not exist, returning `(address(0), false)`. This catch block is intended to handle `OwnerQueryForNonexistentToken` errors, but it also catches other critical failures such as out-of-gas errors, contract pausing, contract upgrades, or other unexpected errors. When `safeOwnerOf()` is called in loops within `getTotalUnclaimedAmounts()` and `_setDividends()`, and gas exhaustion occurs on later iterations, the 63/64 gas rule reserves enough gas for the A26Z contract to complete its remaining computation, allowing the transaction to succeed while silently skipping NFTs that failed due to gas exhaustion or other critical errors. This causes those NFT owners to be excluded from dividend calculations.

&nbsp;

## Internal Pre-conditions

1. Owner needs to call `setDividends()` or `setUpTheDividends()` to execute `_setDividends()` or `getTotalUnclaimedAmounts()`
2. The number of minted NFTs (`nft.getTotalMinted()`) needs to be large enough that iterating through all NFTs consumes significant gas
3. Gas exhaustion needs to occur during `extOwnerOf()` calls on later iterations in the loop
4. The reserved gas from the 63/64 rule needs to be sufficient for the A26Z contract to complete its remaining computation after the gas exhaustion occurs

&nbsp;

## External Pre-conditions

N/A

&nbsp;

## Attack Path

1. Owner calls `setDividends()` or `setUpTheDividends()` to distribute dividends to TVS holders
2. The function calls `_setDividends()` which iterates through all minted NFTs using `nft.getTotalMinted()` as the loop bound
3. For each NFT ID, `safeOwnerOf(i)` is called, which internally calls `nft.extOwnerOf(i)` within a try-catch block
4. During later iterations of the loop, gas exhaustion occurs when calling `extOwnerOf()` on an NFT, or the NFT contract reverts due to experiencing other errors
5. The catch-all `catch {}` block in `safeOwnerOf()` treats this critical failure as a non-existent token and returns `(address(0), false)`
6. The loop continues processing remaining NFTs, and due to the 63/64 gas rule, the reserved gas is sufficient for the A26Z contract to complete its computation
7. The transaction completes successfully, but NFTs that failed due to gas exhaustion or other errors are silently skipped
8. Users who own those skipped NFTs do not receive dividend allocations, causing them to miss out on dividends

&nbsp;

## Impact

Affected TVS holders suffer a complete loss of dividend allocations for their NFTs. When gas exhaustion or other critical errors occur during `extOwnerOf()` calls in later loop iterations, those NFTs are silently excluded from dividend calculations. The transaction completes successfully, giving the false impression that all NFTs were processed, while affected users receive no dividends. 

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation
Implement specific error handling in the catch block to differentiate between non-existent tokens and critical failures.
  