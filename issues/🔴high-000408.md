# [000408] Inadequate Flat Fee Cap Creates Unfair Cross-Decimal Token Pricing
  
  ### Summary

The `_setBidFee` and `_setUpdateBidFee` functions in `FeesManager.sol` are flat fee amounts (not percentage-based) denominated in the project's stablecoin, but use an inadequate upper limit of `100001`. For 6-decimal stablecoins like USDC, this allows fees up to 0.100001 USDC (reasonable), but for 18-decimal stablecoins like USDT, DAI, it permits only 0.000000000000100001 DAI (infinitesimal). This creates massive pricing disparity: users bidding on USDC-based projects pay meaningful fees while users on 18-decimal token projects pay essentially nothing, breaking fee economics and fairness.


### Root Cause

The hardcoded cap of `100001` does not account for varying decimal standards across stablecoins. The check `require(newBidFee < 100001)` treats all stablecoins identically, ignoring that USDC uses 6 decimals while DAI/USDT use 18 decimals, creating a 10^12 magnitude difference in actual fee amounts for the same numeric value.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


**Important context**: `bidFee` and `updateBidFee` are **flat fee amounts** (not basis point percentages) denominated in the project's stablecoin. They are charged per bid/update operation regardless of bid size.

Relevant code excerpts: 
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L84-L92

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L110-L117
```solidity
uint256 constant public BASIS_POINT = 10_000; // Used for mergeFeeRate/splitFeeRate (percentage fees)

function _setBidFee(uint256 newBidFee) internal {
    require(newBidFee < 100001, "Bid fee too high");
    // ... other snippet
}
function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high");
    // ... other snippet

}
```

The decimal disparity problem:

| Stablecoin Type | Decimals | Max Fee (100000) | Human-Readable | Impact |
|----------------|----------|------------------|----------------|--------|
| USDC | 6 | 100000 | 0.1 USDC | Reasonable flat fee |
| USDT, DAI, USDD | 18 | 100000 | 0.000000000000100001 DAI | Effectively zero |

Real-world scenario:
1. **Project A** uses USDC (6 decimals):
   - Owner sets `bidFee = 100000` (0.1 USDC per bid)
   - User Alice bids → pays 0.1 USDC fee
   
2. **Project B** uses DAI (18 decimals):
   - Owner sets `bidFee = 100000` (0.000000000000100001 DAI per bid)
   - User Bob bids → pays 0.000000000000100001 DAI (dust)
   
3. **Result**: Bob pays 12 orders of magnitude less than Alice for the same operation.


### Impact

## Impact
- **Economic unfairness**: Users bidding on 6-decimal stablecoin projects pay ~$0.10 per bid; users on 18-decimal projects pay dust (10^-12 tokens)
- **Fee arbitrage**: Rational users always choose 18-decimal projects to avoid meaningful fees, concentrating activity unfairly
- **Protocol revenue loss**: Owner cannot collect reasonable fees from 18-decimal token projects due to cap being too low

### PoC

_No response_

### Mitigation

### Use fixed higher cap for all tokens
If tracking decimals is too complex, use a much higher universal cap that works for 18-decimal tokens:

```diff
 function _setBidFee(uint256 newBidFee) internal {
-    require(newBidFee < 100001, "Bid fee too high");
+    require(newBidFee < 10e18, "Bid fee too high"); // Cap at 10 tokens (works for 18 decimals)

     uint256 oldBidFee = bidFee;
     bidFee = newBidFee;

     emit bidFeeUpdated(oldBidFee, newBidFee);
 }
```

```diff
 function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
-    require(newUpdateBidFee < 100001, "Bid update fee too high");
+    require(newUpdateBidFee < 10e18, "Bid update fee too high"); // Cap at 10 tokens

     uint256 oldUpdateBidFee = updateBidFee;
     updateBidFee = newUpdateBidFee;

     emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
 }
```
  