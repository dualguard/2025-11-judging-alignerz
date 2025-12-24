# [000171] Pauseability is not enforced on NFT transfers
  
  ### Summary

When the NFT contract is paused the the `mint` and `burn` functions are paused, but the users are still able to perform transfers. As of the terms described in the discord, Low impact is described as `no funds at risk ; Minor incorrect behaviour, state handling issues, etc.`, which applies for this issue. Considering it is at least medium likelihood because every time when the pausing functionality is enforced on the NFT contract, this should be accepted as valid Low/Med severity findings

### Root Cause

The pausing functionality is not enforced on NFT transfers

### Internal Pre-conditions

NFT contract being paused and user transfering the NFTs

### External Pre-conditions

None

### Attack Path

1. The Pausing functionality is enforced on the NFT contract
2. Users transferring the NFT, bypassing the main invariant of pausing which is to stop users from changing the state of the blockchain until certain owner activities are done

### Impact

Users bypass the main invariant of pausing functionality also as described in the discord: `Minor incorrect behaviour`

### PoC

Not needed

### Mitigation

Enforce the pausing functionality on transfers.
  