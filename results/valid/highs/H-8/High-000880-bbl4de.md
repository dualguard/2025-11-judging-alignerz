# [000880] Infinite loop OOG in `getUnclaimedAmounts()` breaks core functionality
  
  ### Summary

In `getUnclaimedAmounts()` we have the following for loop:
```solidity
        for (uint i; i < len;) {
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
```
This loop is implemented incorrectly and will become infinite, reverting with OOG.

### Root Cause

Loop indexing parameter `i` is updated only if `claimedFlows[i] != true && claimedSeconds[i] != 0`. Otherwise, if we enter any of said if statements, we hit `continue` only to keep iterating over that same nft.

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

-

### Impact

Even if `setAmounts()` succeeded once, if cannot be re-called due to OOG once any of the nfts hit the if wrongly implemented if statements. 

### PoC

-

### Mitigation

Increment `i` inside these if statements - or do it unconditionally in the for loop definition.
  