# [000249] Infinite Loop Causes Denial of Service in Dividend Distribution
  
  ### Summary

The `A26ZDividendDistributor::getUnclaimedAmounts()` function contains a loop where the increment statement `(++i)` is placed after `continue` statements. When certain common conditions are met (claimed flows or unclaimed flows with zero claimed seconds), the loop variable `i` never increments, causing an infinite loop. This makes the `::setUpTheDividends()` function run out of gas and revert, permanently breaking the dividend distribution functionality.

### Root Cause

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint256 i; i < len;) {
        if (claimedFlows[i]) continue;  // Skips increment!
        
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;  // Also skips increment!
        }
        
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        
        unchecked {
            ++i;  // Only reached if NEITHER continue is hit!
        }
    }
    unclaimedAmountsIn[nftId] = amount;
    }
```

- The loop incremet `++i` is inside the `unchecked` block at the end
- Two `continue` statements jump back to the loop start before reaching the increment
- Loop variable `i` stays the same value forever
- Loop never exits, consumes all gas

### Internal Pre-conditions

Any NFT must have at least one flow where either:

1. The flow has been fully claimed → `claimedFlows[i] == true`
2. The flow has not been touched yet → `claimedSeconds[i] == 0`

### External Pre-conditions

none

### Attack Path

**Setup:**
- Alice owns NFT-0 with a vesting allocation containing multiple flows
- Flow 0: 1000 tokens, fully claimed → `claimedFlows[0] = true`
- Flow 1: 2000 tokens, active vesting → `claimedFlows[1] = false, claimedSeconds[1] = 5000`

**Steps:**

1. **Owner calls `setUpTheDividends()`**
   - Function executes `_setAmounts()`

2. **`_setAmounts()` calls `getTotalUnclaimedAmounts()`**
   - Iterates through all minted NFTs
   - For each NFT, calls `getUnclaimedAmounts(nftId)`

3. **`getUnclaimedAmounts(0)` is called for Alice's NFT**
   - Loop starts: `for (uint256 i; i < 2;)` (2 flows)
   - **Iteration 1:** `i = 0`
     - Checks: `if (claimedFlows[0])` → **TRUE** (flow is fully claimed)
     - Executes: `continue` → jumps back to loop condition
     - **`i` is still 0** (increment was skipped)
   - **Iteration 2:** `i = 0` (same value!)
     - Checks: `if (claimedFlows[0])` → **TRUE** (still the same flow)
     - Executes: `continue` → jumps back to loop condition
     - **`i` is still 0**
   - **Iteration 3...∞:** Same check, same continue, `i` never increments
   - Loop runs forever consuming gas

4. **Transaction reverts with out-of-gas error**
   - `getUnclaimedAmounts()` never completes
   - `getTotalUnclaimedAmounts()` never completes
   - `setUpTheDividends()` fails completely

5. **All future attempts fail**
   - Every call to `setUpTheDividends()` hits the same infinite loop
   - Dividend distribution system is permanently broken
   - NFT holders can never receive dividends

### Impact

Complete DOS as `::_setUpDividends()` always reverts with out-of-gas.

### PoC

_No response_

### Mitigation

_No response_
  