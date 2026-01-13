# [000475] Infinite Loop in A26ZDividendDistributor::getUnclaimedAmounts Enables Dividend Distribution DoS
  
  ### Summary

The A26ZDividendDistributor::getUnclaimedAmounts(uint256 nftId) uses a for loop with continue statements that do not increment the loop index i.

For many normal TVS states (e.g., claimedSeconds == 0 or claimedFlows == true), the loop never advances and enters an infinite loop, causing out-of-gas reverts.

### Root Cause

1. When claimedFlows[i] == true, the first continue runs with no i++, so i never changes.
2. When claimedFlows[i] == false and claimedSeconds[i] == 0 (fresh flow), the second continue also runs with no i++.
3. The index is only incremented in the last branch, which is reached only if !claimedFlows[i] && claimedSeconds[i] != 0.


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

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L152

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls setUpTheDividends() to initialize dividends.
2. Contract calls _setAmounts() → getTotalUnclaimedAmounts() → getUnclaimedAmounts(nftId).
3. Contract enters for loop in getUnclaimedAmounts() where claimedSeconds[0] == 0 or claimedFlows[0] == true.
4. Loop hits continue without incrementing i, causing an infinite loop and out-of-gas revert.
5. Result: Owner’s dividend setup transactions always revert


### Impact

Dividend setup reverts:
* _setAmounts() calls getTotalUnclaimedAmounts() → getUnclaimedAmounts().
* Any “bad” NFT (very normal state) causes the whole transaction to run out of gas and revert.

### PoC

TODO

### Mitigation

_No response_
  