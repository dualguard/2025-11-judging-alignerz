# [000507] Missing USD Price Conversion Causes Unfair Dividend Distribution Across Different Token Values
  
  ### Summary

The `getTotalUnclaimedAmounts()` and `getUnclaimedAmounts()` functions calculate unclaimed token amounts in raw token quantities without converting to USD values. When different TVS NFTs hold tokens with different USD prices (e.g., Token A = $1, Token B = $5), users holding lower-value tokens receive disproportionately higher dividend shares relative to their actual USD value.

### Root Cause

In `A26ZDividendDistributor.sol` contract
In `getUnclaimedAmounts()`:
```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... code ...
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
@>          amount += amounts[i]; //  Raw token amount, not USD value
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
@>      amount += unclaimedAmount; //  Raw token amount, not USD value
        // ...
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

The function returns raw token amounts without normalizing to USD. The comment `@notice USD value in 1e18` suggests it should return USD values, but the implementation doesn't perform any price conversion. This creates an unfair distribution when the dividend distributor divides stablecoins based on these non-normalized amounts.

Code Snippet-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161


### Internal Pre-conditions

1. Multiple TVS NFTs exist with different underlying tokens.
2. Different tokens have different USD prices.


### External Pre-conditions

1. Protocol supports vesting for multiple token projects (Token A, Token B, etc.).
2. These tokens have different market values.
3. Owner calls `setUpTheDividends()` to distribute stablecoins.


### Attack Path

Taking this for better understanding this issue
**Setup:**
- User A: Has 100 Token A (price = $1 USD) = $100 USD value
- User B: Has 100 Token B (price = $5 USD) = $500 USD value
- Total real value: $600 USD
- Stablecoin to distribute: 600 USDC

**Current (Incorrect) Calculation:**
1. `getTotalUnclaimedAmounts()` runs:
   - User A: 100 tokens (raw amount)
   - User B: 100 tokens (raw amount)
   - Total: 200 tokens

2. Dividend distribution:
   - User A: (100/200) × 600 USDC = 300 USDC
   - User B: (100/200) × 600 USDC = 300 USDC

**Actual Fair Distribution (Should Be):**
1. With USD conversion:
   - User A: 100 × $1 = $100 USD
   - User B: 100 × $5 = $500 USD
   - Total: $600 USD

2. Fair dividend distribution:
   - User A: (100/600) × 600 USDC = 100 USDC
   - User B: (500/600) × 600 USDC = 500 USDC

**Result:** User A receives 300 USDC for $100 worth of tokens, while User B receives only 300 USDC for $500 worth of tokens. User A gets 3x more than fair share.

### Impact

 Fundamentally broken dividend distribution mechanism. Users with low-value tokens are massively overcompensated. Users with high-value tokens are massively undercompensated. Makes the entire dividend system economically unfair and exploitable. Can be gamed by users minting cheap tokens to inflate dividend shares.


### PoC

N/A

### Mitigation

N/A
  