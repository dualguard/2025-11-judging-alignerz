# [000398] Missing Whitelist Check in updateBid()
  
  ### Summary

The `updateBid()` function does not enforce the whitelist restriction that exists in `placeBid()`.
This allows blacklisted or non-whitelisted users, who are not allowed to participate in the bidding to update an existing bid even after their access should have been revoked.

This breaks the project’s intended access-control logic and creates a scenario where restricted users can continue interacting with the protocol.

Same for other functions like:
mergeTVS()
splitTVS()
ClaimTVS()
ClaimTokens() etc

### Root Cause

The protocol enforces whitelist logic in `placeBid()`:

```solidity
if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
}
```

However, the same logic is not implemented in `updateBid()`, even though the function modifies important bidding parameters (amount and vestingPeriod). This omission allows any address with an existing bid, regardless of whitelist status, to continue interacting with the system.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- User is initially whitelisted and places a valid bid.
- Project owner removes the user from whitelist (blacklists them).
- User calls updateBid() → function does not check whitelist.
- User can:
  - Increase their bid amount
  - Extend their vestingPeriod
  - Influence pool allocation outcomes
  - Continue competing in the bidding process

This violates project owner’s control and intended bidding restrictions.


### Impact

- Business Logic Bypass
A user who is supposed to be excluded from bidding can still actively participate.

- Allocation Manipulation
By increasing their bid amount after being blacklisted.

- Unfair Treatment
Users who are removed from whitelist should lose bidding privileges.  

### PoC

_No response_

### Mitigation

Add the same whitelist validation inside updateBid() that exists in placeBid():

```solidity
if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
}
```
  