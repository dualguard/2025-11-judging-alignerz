# [000769] Unbounded NFT scans in dividend setup enable gas DoS via inflated `getTotalMinted()`
  
  
### Summary

Using `for (i = 0; i < nft.getTotalMinted(); ++i)` in `getTotalUnclaimedAmounts` and `_setDividends` will cause a gas based DoS on dividend setup, as any user can inflate `nft.getTotalMinted()` (via repeated splits/merges) until `setUpTheDividends` / `setDividends` becomes too expensive or impossible to execute.


### Root Cause

```solidity
// File: A26ZDividendDistributor.sol

function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}

/// @notice Internal logic that allows the owner to set the dividends for each TVS holder
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

`nft.getTotalMinted()` is lifetime minted count , not active supply. Both loops linearly scan `0 .. getTotalMinted() - 1` every time dividends are set up.  `AlignerzNFT` is minted by vesting (claims, splits, merges), so an attacker can inflate `getTotalMinted()` arbitrarily over time.

There is no cap, pagination, or active IDs only tracking; the cost of running these loops grows monotonically and permanently.



### Internal Pre-conditions

1. Vesting / NFT contracts allow repeated minting of TVS NFTs (e.g. via `splitTVS`, multiple reward projects, etc.).  
2. The dividend distributor’s `nft` reference points to that NFT contract, and `setUpTheDividends` / `setDividends` will be called periodically by an admin.



### External Pre-conditions

1. An attacker holds at least one TVS NFT and is willing to pay some gas / fees to grief the system (e.g. by repeatedly calling `splitTVS`).



### Attack / Failure Path

1. Attacker obtains one or more TVS NFTs.  
2. Attacker repeatedly calls operations that mint additional NFTs (e.g. `splitTVS`) to create many new `tokenId`s; even if some are later burned or merged, **`getTotalMinted()` keeps increasing**.  
3. Each new split permanently increments the upper bound `len` used in both `getTotalUnclaimedAmounts` and `_setDividends`.  
4. Over time, `len` becomes so large that:
   - `getTotalUnclaimedAmounts` and `_setDividends` consume too much gas to fit in a block, or
   - They become economically infeasible to call for a normal admin transaction.  
5. Result: `setUpTheDividends` / `setDividends` becomes practically or strictly **unexecutable**, DoSing the dividend subsystem for all users.



### Impact

The dividend system can be griefed / DoSed by any user who grows `nft.getTotalMinted()` sufficiently:

- Admin can no longer successfully call `setUpTheDividends` / `setDividends` once `len` crosses a certain threshold.
- No new dividend epochs can be set; TVS holders effectively **lose access to future dividends**, even though vesting itself continues to function.

It doesn’t let the attacker steal funds directly, but it can permanently block intended dividend distributions for everyone.

---

### PoC



### Mitigation

Avoid scanning `0 .. getTotalMinted()` every time. Track an active list of dividend-eligible NFT IDs and iterate that instead.

  