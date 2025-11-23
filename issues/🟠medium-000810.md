# [000810] `updateBid()` has no whitelist check
  
  ### Summary

`updateBid()` has no whitelist check unlike `placeBid()`. Therefore if a user has been previously whitelisted and has placed a bid, he will be able to update his bid if unwhitelisted afterwards.

### Root Cause

`updateBid()` has no whitelist check unlike `placeBid()`

### Internal Pre-conditions

User must be whitelisted and then unwhitelisted

### External Pre-conditions

None

### Attack Path

1. Alice places a bid, while she is whitelisted
2. Owner removes Alice from whitelist
3. She can still update her bid

### Impact

Alice will be able to update her bid, even when not whitelisted.

### PoC

_No response_

### Mitigation

_No response_
  