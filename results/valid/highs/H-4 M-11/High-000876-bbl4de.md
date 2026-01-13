# [000876] `dividendsOf` is never reset between dividend rounds
  
  ### Summary

`A26ZDividendDistributor` contract is meant to be used multiple times, for multiple rounds of dividends. Each time, the owner simply calls `setUpTheDividends()`, which updates the distribution amount for each NFT. 

### Root Cause

`dividendsOf[owner].amount` and `dividendsOf[owner].claimedSeconds` are not reset anywhere apart from `claimDividends()`:
```solidity
        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
```
Dividend claim is doable only by the owner of the underlying NFT, which makes it impossible for anyone to reset these values for them, **even** if the vesting period has already been over for a long time. 

### Internal Pre-conditions

- admin sets a new dividend distribution after one was already set ( natural flow ) 

### External Pre-conditions

-

### Attack Path

Malicious user can deliberately **not** claim their distribution at all. After the owner attempts to set the new dividends, that user's old dividend amount will be added to the new one, which effectively let's them steal from other users. 

### Impact

Users that never claimed will be eligible to a larger distribution than they should given that their old `dividendsOf[owner].amount` was incremented (`+=`) by the new distribution amount, without resetting the old one. This effectively lets them steal distributions from other users ( because the total claimable amount per accounting is larger than the actual distribution token balance in the contract ). 

Users that partially claimed last round will either have to claim less than their are eligible to in the future rounds OR not be able to claim at all if `claimedSeconds` from previous round + current `claimedSeconds` is larger than `secondsPassed`. 



### PoC

_No response_

### Mitigation

Reset `dividendsOf[owner]` for all users before setting the new values. This way, it becomes a user mistake if they failed to claim their old dividend ( as it is now part of the new distribution ). 
  