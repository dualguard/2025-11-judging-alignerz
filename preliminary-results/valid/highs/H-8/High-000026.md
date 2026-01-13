# [000026] Infinite-loop in `getUnclaimedAmounts()` permanently bricks dividend accounting
  
  ### Summary

The function `getUnclaimedAmounts()` contains a broken loop structure that fails to increment its index across multiple branches:

```solidity
for (uint i; i < len; ) {
    if (claimedFlows[i]) continue;                // NO i++
    if (claimedSeconds[i] == 0) {                // NO i++
        amount += amounts[i];
        continue;
    }
    uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked { ++i; }
}
```

Because both `claimedFlows[i] == true` **and** `claimedSeconds[i] == 0` lead to `continue` before the increment, the loop **never advances** for any flow that enters those branches.

Crucially, **every vesting flow starts with `claimedSeconds[i] == 0`**, so as soon as even one NFT has a TVS allocation, the function will enter an infinite loop on the very first flow.

This means the dividend system cannot initialize or update under any normal state.

### Root Cause

The loop in `getUnclaimedAmounts()` uses continue in branches that skip the `i++`, causing the index to never advance for initial or claimed flows, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L152.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Dividend setup and updates always revert due to an infinite loop, permanently breaking the entire dividend distribution system once any TVS allocation exists.

### PoC

_No response_

### Mitigation

Consider refactoring the loop to ensure that the loop index (`i`) is incremented on *all* code paths.
  