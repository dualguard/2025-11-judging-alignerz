# [000093] Removed whitelisted users retain protocol access
  
  ### Summary

The missing whitelist validation in `updateBid()`, `claimNFT()`, `claimRefund()`, `claimTokens()`, and NFT management functions will allow users removed from the whitelist to continue participating fully in whitelisted projects, as they will have placed an initial bid while whitelisted, then continue using all protocol functions after being removed from the whitelist.

### Root Cause

In `AlignerzVesting.sol`, only `placeBid()` at lines 708-710 enforces whitelist validation.  All other user-facing functions (updateBid, claimNFT, claimRefund, claimTokens, mergeTVS, splitTVS) lack this check, allowing users who were removed from the whitelist to continue full participation in the project.

(https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L709-L711)

### Internal Pre-conditions

1. Owner needs to enable whitelist for a project via `enableWhitelist()` WhitelistManager.sol:49-51
2. Owner needs to add user to whitelist via `addUserToWhitelist()` WhitelistManager.sol:103-105
3. User needs to place a bid while whitelisted
4. Owner needs to remove user from whitelist via `removeUserFromWhitelist()` WhitelistManager.sol:133-135

### External Pre-conditions

N/A

### Attack Path

1. Owner enables whitelist for project 0 and adds userA to whitelist
2. UserA places bid of 1,000 USDT via `placeBid()` - passes whitelist check 
3. Owner removes userA from whitelist via `removeUserFromWhitelist()` 
4. UserA calls `updateBid()` to increase bid to 100,000 USDT - succeeds because no whitelist check 
5. After project closes, userA calls `claimNFT()` - succeeds because no whitelist check 
6. UserA calls `claimTokens()` to claim vested tokens - succeeds because no whitelist check 
7. UserA can merge/split NFTs and perform all other operations without restriction

### Impact

User removed from the whitelist retain complete access to all protocol fucntionality for projects they previosly participated in. This undermines the whitelist mechanism's purpose.
The owner do not have the abiltiy to revoke access from malicious or unwanted actors from interacting with protocols, as removal from whitelist only prevents place new bids, not all other operation mentioned above.

### PoC

_No response_

### Mitigation

The solutions is to add the same validation that exists in `placeBids()` to all-user facing functions:

1. `updateBid()`
2. `claimNFT()`
3. `claimRefund()`
4. `claimTokens()`
5. `mergeTVS()`
6. `splitTVS()`

note that this validation may be tricker in merge and split becuase these fucntions work accross differnet reward and biding projects
  