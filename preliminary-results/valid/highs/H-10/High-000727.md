# [000727] In `calculateFeeAndNewAmountForOneTVS`  fee is calculated on the whole allocated amount even though funds may have already been claimed
  
  ### Summary

Incorrect fee calculation on the full allocation amount instead of only the remaining tokens, leading to insolvency where the last users cannot claim their entitled tokens.



### Root Cause

In `splitTVS` and `mergeTVS`, the fee is calculated on the full original allocation amount `allocation.amounts` instead of only the unclaimed/remaining tokens. This causes the fee to be extracted from tokens that users have already claimed and withdrawn from the contract.

```solidity
uint256[] memory amounts = allocation.amounts; // Full original allocation
uint256 nbOfFlows = allocation.amounts.length;
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
allocation.amounts = newAmounts;
token.safeTransfer(treasury, feeAmount); // Fee sent to treasury immediately
```

The same issue exists in `mergeTVS`.

### Internal Pre-conditions

1. User А needs to have claimed at least some portion of their vested tokens before splitting/merging
2. Split fee rate needs to be set to a non-zero value (e.g., 200 basis points = 2%)


### External Pre-conditions

_No response_

### Attack Path

1. User receives a vesting NFT with 1000 tokens allocated over 10 days
2. After 5 days (50% of vesting period), user calls `claimTokens()` and receives 500 tokens.
3. User calls `splitTVS()` to split their NFT 50/50 between two recipients
4. Contract calculates fee on full 1000 tokens: `1000 × 2% = 20 tokens`
5. Fee of 20 tokens is immediately transferred to treasury
6. Remaining allocation is reduced to 980 tokens and split 50/50: 490 tokens each
7. Over the remaining 5 days, each recipient can claim: `490 × 5/10 = 245 tokens`
8. Total claimable after split: `245 × 2 = 490 tokens`

**Total accounting:**
- Initially allocated: 1000 tokens
- Already claimed by user: 500 tokens  
- Fee charged: 20 tokens
- Remaining claimable: 490 tokens
- **Total**: 500 + 20 + 490 = **1010 tokens** (but contract only has 1000!)


### Impact

The last person will not be able to claim because there will not be enough tokens in the contract.

### PoC

_No response_

### Mitigation

Modify both `splitTVS` and `mergeTVS` functions to calculate fees only on unclaimed tokens

  