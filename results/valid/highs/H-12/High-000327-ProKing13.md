# [000327] Inverted Token Filter Logic Causes Complete Dividend Misdirection
  
  ### Summary

In [getUnclaimedAmounts()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) uses `==` instead of `!=`, causing the function to return 0 for legitimate token holders and calculate amounts for wrong token holders. This completely breaks dividend distribution - legitimate holders get nothing, wrong token holders get everything.

### Root Cause


```javascript
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
 --->   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  
    // ... 
}
```

The conditional uses equality (`==`) instead of inequality (`!=`), causing the function to:
- Return 0 when the NFT's token **MATCHES** the dividend contract's target token (should continue calculation)
- Continue calculation when the NFT's token **DOESN'T MATCH** (should return 0)


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

it's a functional bug that breaks normal operation:

1. Owner deploys contract for A26Z token
2. System has:
   - NFT 1: 100k A26Z tokens (legitimate)
   - NFT 2: 50k A26Z tokens (legitimate)  
   - NFT 3: 75k DifferentToken (wrong)
3. Owner calls `setUpTheDividends()`
4. `getUnclaimedAmounts()` returns:
   - NFT 1: 0 (should be 100k)
   - NFT 2: 0 (should be 50k)
   - NFT 3: 75k (should be 0)
5. Dividends allocated based on wrong amounts
6. Legitimate holders get 0 USDC, wrong holder gets all USDC


### Impact


- 100% of dividends go to wrong token holders
- 0% of dividends go to legitimate holders
- Affects all dividend distributions


### PoC

_No response_

### Mitigation

Change the equality operator to inequality

  