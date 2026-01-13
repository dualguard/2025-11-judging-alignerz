# [000623] Owner will fail to allocate dividends to legitimate TVS holders due to inverted token comparison in `getUnclaimedAmounts`
  
  ### Summary

Inverted token comparison logic in `A26ZDividendDistributor::getUnclaimedAmounts` will cause zero unclaimed amounts to be calculated for all legitimate TVS holders whose NFTs contain the correct project token, as the function returns 0 when token addresses match instead of when they don't match, resulting in complete failure of the dividend distribution system.

### Root Cause

In [A26ZDividendDistributor.sol:getUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141)  the token comparison uses equality (==) instead of inequality (!=):
```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    //                           ↑  WRONG: Should be != (not equal)
    // This returns 0 when tokens match (should be opposite)
    // ...
}
```
This causes the function to return 0 (no unclaimed tokens) for NFTs that contain the correct project token, while incorrectly calculating amounts for NFTs with different/wrong tokens

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Owner calls setUpTheDividends()

### Impact

The dividend distribution system is non-functional

### PoC

_No response_

### Mitigation

```diff
-   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+   if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  