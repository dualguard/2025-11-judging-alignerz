# [000770] Inverted token validation in `getUnclaimedAmounts` excludes intended TVS NFTs from dividends
  
  
### Summary

Using `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;` in `getUnclaimedAmounts` will cause dividends to be computed only for NFTs whose vesting token does not match the configured `token`, so in the common case (single TVS token) all intended NFTs are filtered out and dividends either never set up or allocated to the wrong set of NFTs.



### Root Cause

In `A26ZDividendDistributor`, the filter that is supposed to *exclude* NFTs whose vesting token doesn’t match the configured TVS token is reversed:

```solidity
// File: A26ZDividendDistributor.sol

/// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
/// @param nftId NFT Id
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
>>  if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; //@audit should be !=

    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;

    for (uint i; i < len;) {
        if (claimedFlows[i]) continue; 
        if (claimedSeconds[i] == 0) { 
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
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

Intended logic:  
  “If this NFT’s `alloc.token` is not equal to the configured `token`, skip it.”

Actual logic:  
  “If this NFT’s `alloc.token` is equal to the configured `token`, skip it.”

So with `token` set to the actual TVS token, **all the correct NFTs are filtered out** and never contribute to `unclaimedAmountsIn`.



### Internal Pre-conditions

1. At least one TVS NFT exists with `vesting.allocationOf(nftId).token == token` (the intended TVS token).  
2. The distributor is configured with `token` equal to that TVS token.



### External Pre-conditions

1. Admin calls `setUpTheDividends()` / `_setAmounts()` / `_setDividends()` expecting to distribute stablecoin dividends to TVS holders.



### Attack / Failure Path

1. Admin configures the dividend distributor with `token = <TVS token>` (normal setup).  
2. When `_setAmounts()` runs, it calls `getTotalUnclaimedAmounts()`, which internally calls `getUnclaimedAmounts(nftId)` for each NFT.  
3. For every NFT whose allocation token matches the configured `token`, `getUnclaimedAmounts` returns `0` due to `if (address(token) == address(alloc.token)) return 0;`.  
4. `totalUnclaimedAmounts` becomes `0` (or very small / only counting unintended tokens).  
5. `_setAmounts()` sets `totalUnclaimedAmounts = 0`, and then `_setDividends()` computes `unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts`, which can lead to **division by zero** and revert, or dividends being assigned only to NFTs of the “wrong” token.



### Impact

- In a single‑token setup (all TVS with the same token):
  - `totalUnclaimedAmounts` is effectively `0`, so dividend setup reverts or is meaningless; no intended TVS holders ever receive dividends.
- In a multi‑token setup:
  - Only NFTs whose `alloc.token != token` are counted; dividends are allocated to “foreign” NFTs while the intended TVS holders are completely ignored.

it makes the system either unusable or allocates funds to the wrong population.



### PoC 




### Mitigation

Fix the token filter condition:

```diff
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+        if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;

```

  