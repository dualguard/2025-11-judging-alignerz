# [000861] Assumption That USDC and USDT Always Equal 1 USD Causes Incorrect Dividend Payouts
  
  ### Summary

In `A26ZDividendDistributor.sol`, the `amount` field of the Dividend struct is documented as representing a USD-denominated amount ("amount in USD in 1e6"). However, within `claimDividends()`, the protocol directly transfers the same numerical amount in USDC or USDT without adjusting for the actual market price of the stablecoin. Since both USDC and USDT can deviate from the $1.00 peg, users may receive a payout whose real USD value is lower than intended.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L15



### Root Cause

The implementation assumes that USDC and USDT will always trade at exactly $1.00 and therefore sends stablecoin amounts equal to the USD-denominated amount field. This assumption fails during depeg events, causing the protocol to underpay users relative to the intended USD value.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L202

Historical examples:
- March 2023: USDC depegged to $0.88 during the Silicon Valley Bank collapse.
- Multiple events: USDT has temporarily fallen to $0.95 and below.



### Internal Pre-conditions

None

### External Pre-conditions

A stablecoin must depeg from $1.00 for the impact to manifest.

### Attack Path

None

### Impact

Users may receive a smaller USD-equivalent amount than intended during periods when USDC or USDT trade below their peg. This leads to an unintended loss of value for claimants and violates the expectation that dividend amounts are USD-denominated.

### PoC

None.

### Mitigation

Use a reliable price feed (e.g., Chainlink) to fetch the real-time USD price of USDC and USDT when calculating claim amounts. Convert the USD-denominated amount into the correct number of stablecoin units to maintain accurate payout value.
  