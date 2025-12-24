# [000547] Dividends Assigned to NFT Holder at Time of Setup Enables Sniping Attack and Causes Loss of Rewards for Legitimate TVS Holders
  
  
## Summary

Distributing dividends to the past NFT owner at the time `_setDividends()` executes, rather than the current holder during the earning period, allows attackers to buy NFTs with high unclaimed amounts right before dividends are set, claim them immediately, and sell the NFT. Legitimate holders who now holds the TVS during the earning period lose their dividends to the attacker.

## Root Cause

In [`A26ZDividendDistributor.sol:214-223`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224), `_setDividends()` assigns dividends to the NFT owner at the time of execution (`dividendsOf[owner].amount += ...`). The function uses `safeOwnerOf(i)` to get the current owner, which reflects ownership at execution time, and set them permanently as the dividends benefactor. This creates a window where an attacker can acquire an NFT just before `setUpTheDividends()` is called and get set as the dividends receiver without having to hold the NFT going forward. 
Contrary to the intended design choice as shown by the comment above `claimDividends`, the claimant of the dividends is not always the holder of the TVS. And the current holder loses the associated dividends to that TVS.
```js
    /// @notice Allows a TVS holder to claim his dividends
    function claimDividends() external {...}
```


## Internal Pre-conditions

1. The owner must call `setUpTheDividends()` to set up dividend distribution
2. There must be NFTs with high unclaimed amounts that are available for purchase
3. The attacker must be able to purchase the NFT before `_setDividends()` executes within the same block or transaction sequence

## External Pre-conditions

None required. The vulnerability triggers based on timing between NFT transfers and dividend setup calls.

## Attack Path

1. An attacker anticipates when the owner will call `setUpTheDividends()`
2. The attacker identifies NFTs with high `unclaimedAmountsIn` values that are available for purchase
3. The attacker purchases these NFTs before `setUpTheDividends()` is called
4. The owner calls `setUpTheDividends()`, which calls `_setDividends()`
5. `_setDividends()` iterates through NFTs and assigns dividends to the current owner (the attacker) at line 218
6. The attacker sells the NFT, as their address have being mapped to the dividends and can extract dividends without holding during the earning period
7. The legitimate TVS holder who holds the NFT during the earning period loses their dividends

## Impact

Legitimate TVS holders who holds NFTs during the earning period lose their dividends to attackers who purchase NFTs right before dividend distribution and sells it afterward. Attackers gain the full dividend amount without holding during the earning period. This creates a perverse incentive where holding NFTs during the earning period is not rewarded, while sniping NFTs just before distribution is profitable.

## Proof of Concept
N/A

## Mitigation
Consider distributing the associated dividends for a TVS to the current holder of the TVS at any point in time.
  