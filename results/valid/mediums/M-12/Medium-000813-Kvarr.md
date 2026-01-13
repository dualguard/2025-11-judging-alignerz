# [000813] Users cannot `claimTokens()` in certain edge cases
  
  ### Summary

`claimTokens()` iterates through all vesting flows for an NFT and calls `getClaimableAmountAndSeconds()` for each.
However, that function contains a hard requirement:
```solidity
require(claimableAmount > 0, No_Claimable_Tokens());
```
If any unclaimed flow produces a zero claimable amount, the entire `claimTokens()` call reverts.

As a result, a user with multiple flows can be blocked from claiming tokens from other flows, even when claimable.

**Consider this scenario:**
1. Alice has merged 2 NFTs, so she has now 1 NFT with 2 flows, one of them has a short `vestingPeriod` and the other one has long `vestingPeriod`.
2. Alice happens to call `claimTokens()` just before the short `vestingPeriod` flow has passed and it leaves a very small unclaimed seconds.
3. Next time she wants to call `claimTokens()`, it will go through the short `vestingPeriod` flow as it is not completely claimed. This calculation rounds down to 0: 
```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;
```
and therefore the `claimTokens()` call reverts.

Alice will not be able to claim unless she splits the NFT again.



### Root Cause

```solidity
require(claimableAmount > 0, No_Claimable_Tokens());
```
Reverts `claimTokens()` instead of skipping flows that have `claimableAmount == 0`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

None

### Impact

Users with multiple flows may become unable to claim.

### PoC

_No response_

### Mitigation

_No response_
  