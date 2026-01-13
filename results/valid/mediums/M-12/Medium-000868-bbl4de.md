# [000868] User may break their allocation's claiming mechanism by mistake
  
  ### Summary

When using the split functionality, amount for each underlying allocation for each NFT is calculated:
```solidity
alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```
if you set percentage low enough or `baseAmounts` is relatively small, it will be truncated to `0`. 

This later breaks claiming mechanism.

### Root Cause

No protection against setting the `amounts[j]` to `0` by mistake. 

### Internal Pre-conditions

User splits to too many NFTs OR leaves too little percentage for one of the split NFTs OR the base amount is very small.

### External Pre-conditions

-

### Attack Path

-

### Impact

`claimTokens()` will always revert when any allocation amount is set to `0` as it calls `getClaimableAmountAndSeconds()` for all flows, in which we have the following check:
```solidity
claimableAmount  = (amount * claimableSeconds) / vestingPeriod;
require(claimableAmount > 0, No_Claimable_Tokens());
```
if `amount = 0` it reverts for any `claimableSeconds`. 

### PoC

-

### Mitigation

Ensure the allocation amount cannot be `0` after splitting.
  