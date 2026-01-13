# [000932] updateBid() Missing Whitelist Validation Enables Unauthorized Participation
  
  ### Summary

The missing whitelist check inside [updateBid() ](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-#L776) will cause unauthorized users to bypass the whitelist restriction for bidding projects, as attackers can update bids even when they are not whitelisted, allowing them to increase their allocation, increase their vesting duration, and participate in projects they were explicitly excluded from.

_(According Code-Author blacklisting a user while bidding is ongoing is a possible scenario)_ 

### Root Cause


In placeBid(), the contract enforces:

```
if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
}
```


However, in [updateBid() ](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-#L776) the function does not repeat this check, allowing:

Previously whitelisted but now removed users who is not meant to increase their allocation to execute updateBid() with no whitelist validation. This is a missing authorization check.


### Internal Pre-conditions

1. Owner enables whitelisting for a bidding project (isWhitelisted[msg.sender][projectId] = true).

2. A whitelisted user has been removed from the whitelist after placing bid.
3. User calls updateBid().


### External Pre-conditions

_No response_

### Attack Path

1. A user who is not whitelisted has an existing bid (e.g., blacklisted after placing bid).

2. The user calls `updateBid(projectId, newAmount, newVestingPeriod).
`
3. The function does not check
require(isWhitelisted[msg.sender][projectId]).

4. User successfully:

* increases their bid amount

* increases vesting duration

* continues participating while disallowed

5. The bidding project receives unauthorized bids that violate whitelist requirements.


### Impact

Unauthorized users can continue increasing their bid despite being unwhitelisted, allowing them to participate in private/whitelisted rounds.



### PoC

_No response_

### Mitigation


Add the missing whitelist check inside [updateBid()](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-#L776):

```
if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
}
```
  