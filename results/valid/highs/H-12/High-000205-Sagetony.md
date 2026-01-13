# [000205] Inverted Token Filter in getUnclaimedAmounts Causes Division by Zero and Complete DoS of Dividend Distribution
  
  ### Summary

In `A26ZDividendDistributor.sol:140`, the token equality check is inverted (== instead of !=), which will cause a complete denial of service for all TVS holders as the owner calling setUpTheDividends() will trigger a division by zero revert because all relevant NFTs are incorrectly filtered out, resulting in totalUnclaimedAmounts = 0.

### Root Cause

In `A26ZDividendDistributor.sol:141` the token filter logic is inverted:

``` solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; // INVERTED
    // ...
}
```

The condition should be != (skip non-matching tokens) but uses == (skip matching tokens). This causes all target TVS NFTs to return 0 unclaimed amounts, making totalUnclaimedAmounts = 0, which then causes division by zero in _setDividends() at line 218.

### Internal Pre-conditions

1. Owner needs to deploy `A26ZDividendDistributor` with _token set to the target TVS token address (e.g., A26Z)
2. At least one TVS NFT needs to exist with the matching token 
3. `stablecoin.balanceOf(address(this))` needs to be greater than 0 (rewards deposited)

### External Pre-conditions

None required.


### Attack Path

1. Owner deposits stablecoins to A26ZDividendDistributor for dividend distribution
2. Owner calls setUpTheDividends() to distribute rewards to A26Z TVS holders
3. `_setAmounts()` calls getTotalUnclaimedAmounts() which iterates all NFTs
4. For each A26Z TVS NFT, getUnclaimedAmounts() is called
5. At line 141, address(token) == address(allocation.token) is true for all A26Z TVSs
6. All relevant NFTs return 0, so totalUnclaimedAmounts = 0
7. `_setDividends()` attempts `unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts`
8. Division by zero → transaction reverts
9. Dividend distribution is permanently bricked

### Impact

The entire dividend distribution mechanism is non-functional. TVS holders who held through the vesting period (the "diamond hands" the contract is designed to reward) cannot receive their stablecoin dividends. Stablecoins deposited for distribution remain stuck in the contract, only recoverable via the owner's `withdrawStuckTokens()` emergency function. This represents a complete failure of the protocol's core value proposition for long-term holders.

### PoC

_No response_

### Mitigation

``` solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // Fix: Use != to skip non-matching tokens (keep matching ones)
    if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;

    IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);
    uint256 len = alloc.amounts.length;

    for (uint256 i; i < len;) {
        if (alloc.claimedFlows[i]) {
            unchecked { ++i; }  // Fix: Add increment before continue
            continue;
        }

        if (alloc.claimedSeconds[i] == 0) {
            amount += alloc.amounts[i];
        } else {
            uint256 claimedAmount = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
            amount += alloc.amounts[i] - claimedAmount;
        }

        unchecked { ++i; }
    }

    unclaimedAmountsIn[nftId] = amount;
}

```
  