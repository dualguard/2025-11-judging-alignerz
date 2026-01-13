# [000211] Dividend Distributor Filters Out All Relevant TVS NFTs Due to Inverted Token Matching Logic
  
  ### Summary

The `getUnclaimedAmounts` function has inverted logic that returns 0 when the DividendDistributor's `token` field matches the TVS NFT's allocated token. This is backwards - the function should calculate unclaimed amounts when tokens match (meaning this distributor tracks this type of TVS), and return 0 when they don't match (meaning this distributor is for a different token). As a result, in the intended configuration where a DividendDistributor is deployed to track A26 token TVS positions and distribute stablecoin dividends, all relevant NFTs are filtered out, `getTotalUnclaimedAmounts` returns 0, and `_setDividends` divides by zero.

### Root Cause

In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:141`, the function has an early return that filters out NFTs:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  // ❌ INVERTED LOGIC
    // ... rest of function calculates unclaimed amounts
}
```

**The problem:** This returns 0 when `token` (the DividendDistributor's tracked token) **matches** the NFT's vesting token. This is backwards.

**Correct logic should be:** Return 0 when they **don't match**, and continue calculating when they **do match**.

**Why this breaks dividends:**

1. DividendDistributor is deployed with `token = A26_TOKEN` (to track A26 TVS positions)
2. TVS NFTs have `allocationOf[nftId].token = A26_TOKEN` (they're vesting A26 tokens)
3. These tokens **match**, so line 141 returns 0 immediately
4. This happens for **all** relevant NFTs (all A26 TVS positions)
5. `getTotalUnclaimedAmounts()` sums up all the zeros → returns 0
6. `_setDividends()` tries to divide by `totalUnclaimedAmounts` (which is 0) → **division by zero revert**

### Internal Pre-conditions

1. DividendDistributor needs to be deployed with the `token` parameter set to the same token that's being vested in the TVS NFTs (the intended configuration)
2. At least one TVS NFT must exist with unclaimed vesting tokens
3. Owner calls `setUpTheDividends()` or `setDividends()` to configure dividend distribution


### External Pre-conditions

_No response_

### Attack Path

1. Protocol deploys DividendDistributor with `token = A26_TOKEN` to track A26 TVS positions
2. Users have TVS NFTs that vest A26 tokens (`allocationOf[nftId].token = A26_TOKEN`)
3. Owner funds the DividendDistributor with stablecoin for dividend distribution
4. Owner calls `setUpTheDividends()` to configure dividends:
   - Calls `_setAmounts()` which calls `getTotalUnclaimedAmounts()`
   - `getTotalUnclaimedAmounts()` loops through all NFTs
   - For each NFT, calls `getUnclaimedAmounts(nftId)`
   - `getUnclaimedAmounts` checks: `if (address(A26_TOKEN) == address(A26_TOKEN)) return 0;`
   - Returns 0 for **every single relevant NFT**
   - `getTotalUnclaimedAmounts()` returns 0
5. Then calls `_setDividends()` which attempts to calculate:
   ```solidity
   dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
   ```
   - `totalUnclaimedAmounts = 0` → **division by zero revert**
6. Transaction reverts, dividends can never be configured

### Impact

- **All TVS holders** - cannot receive dividends from their vesting positions

### PoC

_No response_

### Mitigation

_No response_
  