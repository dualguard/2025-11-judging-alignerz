# [000123] Repeated `setUpTheDividends` calls can over-allocate dividends beyond available stablecoin reserves
  
  ### Summary

The A26ZDividendDistributor contract allows multiple calls to `setUpTheDividends()` that cumulatively increase users’ recorded dividend entitlements without isolating or resetting previous rounds. This can cause total theoretical claims to exceed the actual stablecoin balance in the contract. Severity is low because only the owner can trigger these flows and is generally a trusted actor which reduces the likelihood as this will only occur trough an error they make, not for malicious reasons.

### Root Cause


Dividend setup is driven by:

```solidity
function setUpTheDividends() external onlyOwner {
    _setAmounts();
    _setDividends();
}
```

`_setAmounts()` always snapshots the full current stablecoin balance:

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}
```

`_setDividends()` then adds new entitlements on top of existing ones:

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

Because `stablecoinAmountToDistribute` includes reserves from previous rounds and `dividendsOf[owner].amount` is accumulated with `+=`, repeated calls can double-count previously reserved but unclaimed stablecoins. Over time, the sum of all users’ `dividendsOf[*].amount` may exceed the actual stablecoin balance backing them.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. Contract holds 1,000 stablecoins; `setUpTheDividends()` is called once, distributing entitlements summing to 1,000 units across users as `dividendsOf[owner].amount`.
2. No user has yet claimed; contract still holds 1,000 stablecoins.
3. Owner deposits an additional 500 stablecoins and calls `setUpTheDividends()` again.
4. `_setAmounts()` now uses `stablecoinAmountToDistribute = 1_500`, and `_setDividends()` adds a second layer of entitlements based on the full 1,500, on top of the existing `dividendsOf[*].amount`.
5. Total recorded entitlements now represent more than 1,500 units, even though only 1,500 are actually held, allowing early claimants to receive full payouts while later claimants may fail due to insufficient balance.

### Impact


The likelihood of this issue is reduced because only the `onlyOwner` address can call `setUpTheDividends()`, `setAmounts()`, and `setDividends()`. In a typical deployment, the owner is expected to be a trusted protocol operator. Therefore this is a low severity issue 

### PoC

_No response_

### Mitigation


Consider making the dividend rounds explicitly stateful so that each round accounts only for newly added reserves or resets prior allocations. 

One approach could be to introduce an epoch identifier for each distribution round and store dividend entitlements per epoch, or to restrict `setUpTheDividends()` to a single call per configured distribution, with clear operator documentation to avoid repeated reconfiguration on the same pool of funds.
  