# [000307] Removed whitelist bidder still can update his bid by calling updateBid() function
  
  ### Summary

The `AlignerzVesting.sol` contract has a mechanism to whitelisting bidders who will participate in a specific project. However, this check isn't implemented across the board. The check is only performed in the `placeBid()` function, but not in the `updateBid()` function.

This means if a bidder's whitelist status is revoked after executing a `placeBid()` function, they can still update their bids by calling `updateBid()` function.

### Root Cause

In [AlignerzVesting.sol:741](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741) missing check whitelist status for bidder

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

Consider scenario below :

1. Alice as whitelist bidder place a bid by calling `placeBid()` function
2. Whitelist status for Alice got revoked
3. Alice still can update her bid by calling `updateBid()` function even project require whitelist status

### Impact

The application of checks for the whitelist that is not comprehensive causes inconsistency in the bidding process.

### PoC

_No response_

### Mitigation

Consider applied whitelist status check for bidder on `updateBid()` function
  