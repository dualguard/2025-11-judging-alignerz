# [000936] Blacklisted users can updateBid
  
  ### Summary

The `AlignerzVesting::updateBid` lacks whitelist validation, allowing users who have been removed from the whitelist to continue increasing their bid amounts and vesting periods.

### Root Cause

In [AlignerzVesting.sol:741-776](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L776) the `updateBid`  only checks that the project is active and the bid exists, but never validates whether `msg.sender` is still whitelisted for the project. Compare this to placeBid which correctly enforces whitelist status: 

The placeBid function has the whitelist check at lines 709-711:
```solidity
if (isWhitelistEnabled[projectId]) {  
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());  
}
```
But `updateBid` completely omits this validation, allowing removed users to update their existing bids .


### Impact

Blacklisted address has whitelisted address priviledge 



### Mitigation

Add the same whitelist validation to updateBid that exists in placeBid
  