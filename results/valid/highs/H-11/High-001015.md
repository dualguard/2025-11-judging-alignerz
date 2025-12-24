# [001015] Users could lose significant dividends due to rounding down issue
  
  ### Summary

The [claimable](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L201) dividends amount is calculated as: 
```solidity
claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
```
When dividends are paid in a stablecoin with 6 decimals (eg. [USDC](https://arbiscan.io/address/0xaf88d065e77c8cC2239327C5EDb3A432268e5831#readProxyContract#F11)) and the `vestingPeriod` = 3 months = `7_776_000` seconds it means claimers will loose up to `7_776_000` USDC on every claim due to rounding down. 

### Root Cause

The stablecoin used within the protocol are  USDT and USDC, as mentioned in  contest [README](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L201) and     code [comments](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L15) - `amount in USD in 1e6`. 
A realistic dividends `vestingPeriod` is 3 months as [mentioned](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L23-L24) in comments:
```solidity
    /// @notice length of the vesting period (usually 3 months)
    uint256 public vestingPeriod;
```
When the stablecoin used in `A26ZDividendDistributor` has 6‑decimal precision, and the `vestingPeriod` is 3 months (`7_776_000` seconds), the `claimableAmount` calculation may round inconsistently. As a result, up to `7_776_000` USDC wei can be lost (up to more than 7 USD value) on each `claimDividends()` call. 

```solidity
    function claimDividends() external {
//...
         //@audit rounding down, totalAmount is in 6 decimals precision 
        // claimDividends = 3 months = 90 * 24 * 3600 = 7,776,000
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
//...
```

### Internal Pre-conditions

1. The stablecoin used within dividends distributor has low decimals precision (< 18 decimals). 

### External Pre-conditions

None

### Attack Path

1. Admin transfer USDC to `A26ZDividendDistributor` and calls `setUpTheDividends()` to set the dividends  distribution details. 
- Following variables are set: [stablecoinAmountToDistribute](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L208) (6 decimal precision), [unclaimedAmountsIn[nftId]](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L160) (18 decimal precision), [totalUnclaimedAmounts](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L209) (18 dec precision). 
- the dividends amount for each holder is [calculated](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218C26-L218C134) as following:
```solidity
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```
It results that `dividendsOf[owner].amount` has 6 decimal precision.

2. Users calls `claimDividends()`. `claimableAmount` rounds down and and users loose up to 7.776 USDC on each claim.


### Impact

Users receive up to 7.776 USD less in dividends with each claim.

### PoC

_No response_

### Mitigation

The value loss on dividends claiming is significant when the stablecoin used has low decimal precision. 
If, by example, the stablecoin used had 18 decimal precision, the rounding down loss would be insignificant: 7.776 *e6 loss vs 1e18. This may be considered dust amount imo. 

Consider enforcing the stablecoin used in `A26ZDividendDistributor` has 18 decimals precision. 
Additionally, ensure `vestingPeriod` does not surpass a max value (eg. 6months = 2 * 7_776_000s), to cap the rounding error. 
  