# [000560] Unbounded NFT Iteration in Dividend Distribution Functions Causes Permanent Out-of-Gas Failures
  
  

## Summary

Unbounded iteration over all minted NFTs in `_setDividends` and `getTotalUnclaimedAmounts` will cause permanent out-of-gas failures for the protocol and users as an attacker can repeatedly split NFTs to mint an arbitrary number of new NFTs, increasing the total minted count beyond block gas limits.

```js
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            // ...
        }
        // ...
    }
```
&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:127-136`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) and [`A26ZDividendDistributor.sol:214-223`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216-L222), the functions `getTotalUnclaimedAmounts()` and `_setDividends()` iterate over all NFTs from `0` to `nft.getTotalMinted()` without bounds. The choice to iterate over the entire minted NFT collection is a mistake because `getTotalMinted()` can grow unboundedly through the `splitTVS()` function in `AlignerzVesting.sol`, which mints new NFTs for each split operation. As the total minted count increases, the gas cost of these iterations will eventually exceed the block gas limit, making these functions permanently unusable.

&nbsp;

## Internal Pre-conditions

1. An attacker needs to own at least one NFT to initiate the attack
2. The attacker needs to call `splitTVS()` repeatedly to split their NFTs into multiple parts, where each split operation mints `n-1` new NFTs (where `n` is the number of parts in the split)
3. The total number of minted NFTs (`nft.getTotalMinted()`) needs to reach a threshold where iterating over all NFTs exceeds the block gas limit (approximately several thousand NFTs, depending on gas costs per iteration)

&nbsp;

## External Pre-conditions

None required. The attack can be executed entirely within the protocol without external dependencies.

&nbsp;

## Attack Path

1. Attacker acquires one or more NFTs through normal protocol operations
2. Attacker calls `splitTVS()` with a large array of percentages to split a single NFT into many parts (e.g., 100 parts), which mints 99 new NFTs
3. Attacker repeats step 2 multiple times, splitting the newly minted NFTs into even more parts, exponentially increasing the total minted count
4. Once `nft.getTotalMinted()` reaches a sufficiently large number (e.g., 5,000+ NFTs), any call to `getTotalUnclaimedAmounts()` or `_setDividends()` will exceed the block gas limit and revert
5. The protocol owner can no longer call `setUpTheDividends()`, `setAmounts()`, or `setDividends()` to distribute dividends to TVS holders
6. TVS holders cannot receive their dividend distributions, and the dividend distribution mechanism becomes permanently disabled

&nbsp;

## Impact

The protocol and TVS holders cannot execute dividend distribution functions. The protocol owner cannot set up or update dividend distributions, and TVS holders cannot claim dividends. The griefing attack permanently disables core protocol functionality. The loss is the total value of undistributed dividends locked in the contract, which cannot be distributed due to the out-of-gas failures.

&nbsp;

## Proof of Concept

N/A

&nbsp;

## Mitigation

Implement pagination or batch processing to limit iteration per transaction:
1. Add pagination parameters to `getTotalUnclaimedAmounts()` and `_setDividends()` to process NFTs in batches (e.g., 100-200 NFTs per call)
2. Track the last processed NFT index in storage and allow the owner to call these functions multiple times to process all NFTs across multiple transactions


  