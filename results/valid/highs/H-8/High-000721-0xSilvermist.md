# [000721] Missing loop increment before `continue` statements in `getUnclaimedAmounts` causes infinite loop
  
  ### Summary

The `getUnclaimedAmounts` function contains two `continue` statements that skip to the next loop iteration without incrementing the loop counter `i`. This causes infinite loops when these conditions are met, resulting in out-of-gas errors and preventing dividend calculations.

### Root Cause

In the `getUnclaimedAmounts` function, the loop uses the pattern `for (uint i; i < len;)` with manual incrementing at the end of the loop body. However, two `continue` statements exit early without reaching the increment, causing `i` to remain unchanged and creating an infinite loop.

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint i; i < len;) {
->        if (claimedFlows[i]) continue;  // ← i never increments, infinite loop!
        
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
->            continue;  // ← i never increments, infinite loop!
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

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Alice holds NFT 5 with 2 flows: Flow 0 is fully claimed (`claimedFlows[0] = true`), Flow 1 is active
2. Owner calls `getTotalUnclaimedAmounts()` which internally calls `getUnclaimedAmounts(5)`
3. Loop starts with `i = 0`:
   - Check: `claimedFlows[0] = true` → executes `continue`
   - Loop restarts without incrementing `i`
4. Loop iteration 2 with `i = 0`:
   - Check: `claimedFlows[0] = true` → executes `continue` again
   - Loop restarts without incrementing `i`
5. This repeats thousands of times until all gas is consumed
6. Transaction reverts with "out of gas"
7. Owner cannot calculate total unclaimed amounts
8. Dividend distribution system is completely blocked

### Impact

The dividend distribution system becomes completely unusable whenever any NFT has a fully claimed flow or a fresh unclaimed allocation. All calls to calculate unclaimed amounts result in out-of-gas failures, preventing owners from setting up or distributing dividends.

### PoC

_No response_

### Mitigation

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint i; i < len;) {
-       if (claimedFlows[i]) continue;
+       if (claimedFlows[i]) {
+           unchecked { ++i; }
+           continue;
+       }
        
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
-           continue;
+           unchecked { ++i; }
+           continue;
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
  