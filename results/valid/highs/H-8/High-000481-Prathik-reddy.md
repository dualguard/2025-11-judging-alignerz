# [000481] Out of Gas in getUnclaimedAmounts() function
  
  ### Summary

The `A26ZDividendDistributor::getUnclaimedAmounts()` function contains an infinite loop vulnerability. Two `continue` statements skip the loop counter increment (`++i`), causing the loop to never increment if triggered.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L149

### Root Cause

Loop counter `i` is only incremented in the `unchecked` block at the end of the loop. Two `continue` statements bypass this increment:

1. `if (claimedFlows[i]) continue;`
2. `if (claimedSeconds[i] == 0) { ... continue; }`

When either condition is true, the loop jumps to the next iteration without incrementing `i`, causing it to remain at the same index indefinitely.

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
            ...
            ...

            for (uint i; i < len;) {
                if (claimedFlows[i]) continue;
                if (claimedSeconds[i] == 0) {
                    amount += amounts[i];
                    continue;
            }

            ...
            ....
}
```

### Internal Pre-conditions

Vesting allocation exists with at least one index where:
   `claimedFlows[i] == true`, OR
   `claimedSeconds[i] == 0`

### External Pre-conditions

NA

### Attack Path

Logical issue. No attack path

### Impact

All functions that internally use the `getUnclaimedAmounts()` function reverts.
Dividend system halts, owner cannot set up or claim dividends.

### PoC

NA

### Mitigation

Increment `i` before each `continue` statement:

```solidity
for (uint i; i < len;) {
    if (claimedFlows[i]) {
        unchecked { ++i; }
        continue;
    }
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        unchecked { ++i; }
        continue;
    }
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked {
        ++i;
    }
}
```
  