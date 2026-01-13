# [000128] Public `getUnclaimedAmounts` can desync `unclaimedAmountsIn` from `totalUnclaimedAmounts` and strand dividends
  
  ### Summary

The A26ZDividendDistributor contract exposes `getUnclaimedAmounts(uint256)` as a public, state-mutating function that updates per‑NFT unclaimed amounts independently of the aggregated `totalUnclaimedAmounts`. When `setAmounts()` and `setDividends()` are invoked separately, external callers can change `unclaimedAmountsIn` after the snapshot in `setAmounts()` but before `_setDividends()` runs. Severity is high because this desynchronization can cause some NFTs to receive less dividends than intended

### Root Cause

Dividend setup uses a two-step pattern:

```solidity
function setUpTheDividends() external onlyOwner {
    _setAmounts();
    _setDividends();
}

function setAmounts() public onlyOwner {
    _setAmounts();
}

function setDividends() external onlyOwner {
    _setDividends();
}
```

The snapshot of total unclaimed TVS is taken in `_setAmounts()`:

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}

function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}
```

`getTotalUnclaimedAmounts()` calls the public `getUnclaimedAmounts(uint256 nftId)` for each NFT, which both returns a value and writes to `unclaimedAmountsIn[nftId]`:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    for (uint256 i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked { ++i; }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

Dividend allocation in `_setDividends()` relies on `totalUnclaimedAmounts` as a denominator and `unclaimedAmountsIn[i]` as the per-NFT numerator:

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount +=
                (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        }
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```

The core issue is that `getUnclaimedAmounts()` is:

- Publicly callable by anyone.
- State‑mutating, because it re-writes `unclaimedAmountsIn[nftId]`.

`totalUnclaimedAmounts` is only recalculated when `_setAmounts()` is called, and it is not updated when outside callers invoke `getUnclaimedAmounts()`.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. `totalUnclaimedAmounts` is fixed by `_setAmounts()` as a snapshot of unclaimed TVS at some time.
2. Vesting state changes in `AlignerzVesting` (e.g. `claimTokens()` adjusts claimedSeconds and claimedFlows).
3. An attacker (or any third party) calls `getUnclaimedAmounts(nftId)` for specific NFTs, updating `unclaimedAmountsIn[nftId]` to a lower current unclaimed value, **without** updating `totalUnclaimedAmounts`.
4. `_setDividends()` uses stale `totalUnclaimedAmounts` and modified `unclaimedAmountsIn[i]`, leading to per-NFT allocation ratios that no longer match the snapshot of the overall pool.

### Impact

This issue leads to a loss of yield for users and is therefore a high severity issue

### PoC

Assume there are multiple NFTs with allocations, and the owner initiates dividend setup by calling:
   ```solidity
   setAmounts();  // calls _setAmounts -> totalUnclaimedAmounts = snapshot
   ```
2. `getTotalUnclaimedAmounts()` computes `totalUnclaimedAmounts` and sets `unclaimedAmountsIn[nftId]` for all NFTs based on vesting state at that time.
3. Before the owner calls `setDividends()`, some holders claim part of their TVS via `AlignerzVesting.claimTokens(projectId, nftId)`, reducing their actual unclaimed TVS.
4. An attacker (or any third party) calls:
   ```solidity
   getUnclaimedAmounts(nftId1);
   getUnclaimedAmounts(nftId2);
   ```
   to recompute `unclaimedAmountsIn` for a subset of NFTs based on the new lower unclaimed state, while `totalUnclaimedAmounts` remains at the old, higher snapshot.
5. The owner then calls:
   ```solidity
   setDividends();  // calls _setDividends()
   ```
6. `_setDividends()` now allocates dividends using:
   ```solidity
   dividendsOf[owner].amount += unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts;
   ```
   For NFTs whose `unclaimedAmountsIn` were recomputed downward, the numerator is smaller but the denominator is still the old, larger total. These NFTs receive a smaller fraction of `stablecoinAmountToDistribute` than intended for the original snapshot, leaving some of the stablecoin undistributed, which later can only be withdrawn via `withdrawStuckTokens()`.

No mechanism prevents repeated recalculations of `unclaimedAmountsIn` between `setAmounts()` and `setDividends()`, and nothing ties those two operations together unless `setUpTheDividends()` is used atomically.


### Mitigation

Consider making `getUnclaimedAmounts()` a pure view-style helper (or restricting who can mutate `unclaimedAmountsIn`)
  