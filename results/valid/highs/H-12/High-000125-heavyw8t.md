# [000125] Incorrect token comparison in `getUnclaimedAmounts` breaks dividend setup and can cause division by zero
  
  ### Summary

The A26ZDividendDistributor contract incorrectly skips unclaimed amount calculation whenever the configured `token` matches the token stored in the vesting allocation, returning zero instead of the true unclaimed amount. This causes `getTotalUnclaimedAmounts()` to compute `totalUnclaimedAmounts` as zero for the main configured token, which in turn leads to division-by-zero reverts in `_setDividends()`.

### Root Cause

Dividend setup relies on three core functions:

```solidity
function setUpTheDividends() external onlyOwner {
    _setAmounts();
    _setDividends();
}

function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}
```

`getTotalUnclaimedAmounts()` aggregates unclaimed amounts per NFT:

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}
```

The per-NFT unclaimed amount calculation is implemented in `getUnclaimedAmounts()`:

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
        unchecked {
            ++i;
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

Key observations:

- The line:
  ```solidity
  if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
  ```
  short-circuits the function for NFTs whose `vesting.allocationOf(nftId).token` matches the distributor’s configured `token`.
- For a typical deployment, `token` is intended to be “the token inside the TVS” and will match the vesting allocation token for the NFTs that this distributor is meant to handle. In that case, `getUnclaimedAmounts()` returns `0` for every relevant NFT and never populates `unclaimedAmountsIn[nftId]`.
- `getTotalUnclaimedAmounts()` therefore sums only zeros for those NFTs, yielding `_totalUnclaimedAmounts == 0`. `_setAmounts()` then sets:
  ```solidity
  totalUnclaimedAmounts = 0;
  ```
- `_setDividends()` uses `totalUnclaimedAmounts` as the denominator:

  ```solidity
  function _setDividends() internal {
      uint256 len = nft.getTotalMinted();
      for (uint256 i; i < len;) {
          (address owner, bool isOwned) = safeOwnerOf(i);
          if (isOwned) {
              dividendsOf[owner].amount +=
                  (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
          }
          unchecked {
              ++i;
          }
      }
      emit dividendsSet();
  }
  ```

  In Solidity 0.8.29, dividing by zero reverts with a panic whenever `totalUnclaimedAmounts == 0` and there is at least one owned NFT for which the `if (isOwned)` branch executes.

As a result, for NFTs whose allocations use the distributor’s configured `token`, `getUnclaimedAmounts()` always returns zero, `totalUnclaimedAmounts` is zero, and `_setDividends()` will revert, making `setUpTheDividends()` and `setDividends()` unusable in practice.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The distributor is deployed with `token` set to the same token used in `vesting.allocationOf(nftId).token` for the relevant TVSs.
2. A set of NFTs representing TVSs exists, with non-zero unclaimed amounts (`allocationOf(nftId).amounts` populated).
3. The owner calls:
   ```solidity
   setUpTheDividends();
   ```
4. `_setAmounts()` executes:
   - `stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));`
   - `totalUnclaimedAmounts = getTotalUnclaimedAmounts();`
5. In `getTotalUnclaimedAmounts()`:
   - For each owned NFT ID, `getUnclaimedAmounts(nftId)` is called.
   - Each call immediately returns 0 due to:
     ```solidity
     if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
     ```
   - `_totalUnclaimedAmounts` remains 0 and is returned.
6. `_setAmounts()` sets `totalUnclaimedAmounts = 0`.
7. `_setDividends()` runs and for the first owned NFT executes:
   ```solidity
   dividendsOf[owner].amount +=
       (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
   ```
8. Because `totalUnclaimedAmounts == 0`, this division reverts, causing `setUpTheDividends()` (and `setDividends()`) to always fail, preventing any dividend allocation via the intended mechanism.

### Impact

Because this affects a core feature of the contract (dividend distribution) and results in a persistent DoS this is medium severity issue 


### PoC

_No response_

### Mitigation


Consider correcting the token comparison logic in `getUnclaimedAmounts()` to ensure that unclaimed amounts are computed for the token the distributor is meant to support. If the distributor is intended to operate only when `vesting.allocationOf(nftId).token` matches its configured `token`, the early return should be inverted or removed. For example
  