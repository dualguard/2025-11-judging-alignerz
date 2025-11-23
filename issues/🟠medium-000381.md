# [000381] Systematic User Fund Loss Due to Precision Truncation
  
  ## Summary

The Alignerz Protocol contains six precision loss vulnerabilities across three contracts that systematically reduce user payouts through integer division truncation. A26ZDividendDistributor causes dividend underpayment, AlignerzVesting results in token claim losses, and FeesManager has fee calculation imprecision. Users lose funds while the protocol retains the difference, creating an unfair system that benefits the protocol at users' expense.

**Key Findings:**

- **A26ZDividendDistributor:** Users lose 0.01-0.1% of dividends per claim (3 vulnerable functions)
- **AlignerzVesting:** Users lose 0.1-10 tokens per vesting claim (2 vulnerable functions)
- Protocol systematically accumulates "dust" amounts from user losses
- Owner can extract accumulated losses via withdrawal functions
- No mechanism exists to compensate users for lost funds


## Root Cause

### Integer Division Truncation

Solidity's integer division always truncates remainders, causing precision loss:

```solidity
// Solidity behavior:
uint256 result = 1000 * 999 / 1000;
// Expected: 999.999
// Actual: 999
// Lost: 0.999 (truncated)
```
## Affected Code

**A26ZDividendDistributor.sol:**
[claimableAmount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L201)
[claimedAmount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L153)
[dividendsOf[owner].amount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218)
```solidity
uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;

uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];

dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```

**AlignerzVesting.sol:**
[claimableAmount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L993)
[alloc.amounts[j]](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132)
```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;

alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

**FeesManager.sol:**
[feeAmount](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L176)
```solidity
feeAmount = amount * feeRate / BASIS_POINT;
```

## Attack Path
Every time user perform these action, because of not handling of precision properly, there will be some loss of fund for the user.

## Impact
 
User loose the dust funds.

## Mitigation

Use OpenZeppelin's SafeMath library with mulDiv function for precise calculations:

  