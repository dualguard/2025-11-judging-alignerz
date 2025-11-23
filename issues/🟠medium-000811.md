# [000811] Whitelist is not enforced for NFT transfers
  
  ### Summary

`placeBid()` requires `msg.sender` to be whitelisted, so only whitelisted users are supposed to claim NFTs for the specific `projectId`.

However, NFT transfers are not restricted by the whitelist. This allows a whitelisted user to transfer their NFT to a non-whitelisted user.

Since dividend calculations are based on NFT ownership, the non-whitelisted recipient can participate in dividend distributions, bypassing the intended access control.

### Root Cause

There is no whitelist enforcement on NFT transfers.

### Internal Pre-conditions

There must be a whitelist for a specific projectId

### External Pre-conditions

None

### Attack Path

1. Alice is whitelisted, places a bid and claims an NFT.
2. Alice transfers her NFT to Bob, who is not whitelisted.
3. Bob can now participates in dividend distributions.

### Impact

Non-whitelisted users can participate in dividend distributions.

### PoC

_No response_

### Mitigation

_No response_
  