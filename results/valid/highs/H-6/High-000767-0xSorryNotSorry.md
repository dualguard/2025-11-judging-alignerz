# [000767]  `nft.getTotalMinted()` will hit OOG
  
  

### Summary

Unbounded iteration from `0` to `nft.getTotalMinted()` in the dividend distributor creates a gas based DOS risk: any user can inflate `getTotalMinted()` through repeated NFT mints (for example via splits), making `_setAmounts`, `_setDividends`, and `setUpTheDividends` too expensive or impossible to execute.

### Root Cause

In `A26ZDividendDistributor`, both the total unclaimed amount computation and the dividend assignment loop iterate over all ever-minted IDs:

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned)
            dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```

`nft.getTotalMinted()` is a lifetime counter that only increases. Every additional mint, including those created via `splitTVS`, permanently increases the loop bound. Burned NFTs are skipped via `safeOwnerOf`, but they still contribute to `len` and must be iterated over.

### Internal Pre-conditions

1. The NFT contract used by the dividend distributor exposes `getTotalMinted()` as an ever-increasing count of minted token IDs.  
2. Vesting logic (claims, splits, merges) can mint additional NFTs without a strict upper bound, particularly through functions such as `splitTVS`.

### External Pre-conditions

1. The dividend distributor is configured against the TVS NFT contract and used in production.  
2. An operator periodically calls `setUpTheDividends`, `setDividends` or equivalent functions that trigger `_setAmounts` and `_setDividends`.

### Attack Path

An attacker acquires at least one TVS NFT and repeatedly calls NFT-minting paths (for example `splitTVS`) to create many additional token IDs. Each mint increases `getTotalMinted()`, which directly increases the number of iterations performed by `getTotalUnclaimedAmounts` and `_setDividends`. At some point, the gas required to execute these loops exceeds the block gas limit or becomes economically prohibitive. Once this threshold is crossed, calls to `setUpTheDividends` or `setDividends` consistently revert due to out-of-gas, effectively blocking any new dividend epochs for all holders.

### Impact

The dividend subsystem can be permanently or intermittently disabled by inflating `nft.getTotalMinted()`. Dividend setup becomes uncallable in practice, so TVS holders cannot receive stablecoin dividends even though vesting itself remains functional. This constitutes a high-severity griefable denial-of-service within the dividend module.

### POC


### Mitigation

A more robust design would avoid iterating from `0` to `getTotalMinted() - 1` for every dividend epoch. Possible approaches include maintaining a bounded set of dividend-eligible NFT IDs, snapshotting a fixed list or range of IDs for each epoch and processing it in pages, or imposing limits on the number of splits or active dividend-eligible NFTs so that the scan remains within predictable gas bounds.
  