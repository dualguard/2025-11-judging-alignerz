# [000024] Users can receive more dividends than intended
  
  ### Summary

The _setDividends function distributes stablecoinAmountToDistribute proportionally across all TVS holders based on unclaimedAmountsIn[nftId]:
```solidity
/// @notice Internal logic that allows the owner to set the dividends for each TVS holder
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount += (
                unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
            );
        }
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}

```

`stablecoinAmountToDistribute` and `totalUnclaimedAmounts` are recomputed via `_setAmounts()` and can be updated multiple times (e.g. after changing startTime, vestingPeriod, sending more stablecoin, or TVS state changing in vesting). If the owner calls `setUpTheDividends()` / `setDividends()` multiple times over the lifetime of the contract, each call adds a new pro-rata allocation on top of whatever was already assigned, even if the user’s underlying TVS position has not changed proportionally.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-0xAlix2/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218

`_setDividends` accumulates into `dividendsOf[owner].amount (+=)` on each call instead of resetting or recomputing from a clean baseline, so repeated dividend setups compound allocations instead of replacing them.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner configures dividends and calls setUpTheDividends() once – all current holders get a proportional dividendsOf[owner].amount.

2. Later, the owner adjusts startTime / vestingPeriod or deposits more stablecoin and calls setUpTheDividends() again.

3. _setDividends distributes the new stablecoinAmountToDistribute pro-rata based on the updated unclaimedAmountsIn, adding to existing dividendsOf[owner].amount rather than recalculating from scratch.

4. Users end up receiving dividends from both the old and the new snapshot logic, which may overshoot the intended total and distort relative rewards.

### Impact

Dividends can be over-allocated or misallocated when `_setDividends` is called multiple times.

### PoC

_No response_

### Mitigation

Ensure _setDividends writes fresh values instead of accumulating:
```solidity
if (isOwned) {
    dividendsOf[owner].amount = (
        unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
    );
}
```
  