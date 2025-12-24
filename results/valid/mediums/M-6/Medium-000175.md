# [000175] Users who placed a bid before being blacklisted can still update their bid, claim NFT and rewards after being blacklisted.
  
  ### Summary

`placeBid` function allows users to place a bid for a given bidding project. This function includes the following check:

```solidity
       if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
```
This means if a given project allowed the whitelist feature, users that want to place a bid for this project need to be whitelisted.

The problem arises because `updateBid`, `claimNFT`, `claimRefund` and `claimTokens` never check for the whitelist status of the user.


### Root Cause

The root cause is a lack of check in important functions of the protocol. If the user was initially whitelisted, placed a bid and got blacklisted after that, he shouldn't be allowed to update his bid and claim rewards.


### Impact

The impact of this issue is medium as it results in blacklisted users still allowed to interact with the contract, update bids and claim rewards.

### Mitigation

Make sure to add the necessary checks in  `updateBid`, `claimNFT`, `claimRefund` and `claimTokens` functions.
  