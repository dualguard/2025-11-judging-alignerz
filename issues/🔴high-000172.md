# [000172] Too many flows will make it impossible for a user to call `claimTokens` function
  
  ### Summary

When calling [`Vesting::claimTokens()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) function, the gas it will cost is strictly dependent on the number of flows the corresponding NFT supports. If the NFT has too many flows, the function will exceed the block gas limit, hence leading to permanent loss of funds for the user.
Having too many flows on an NFT can be achieved by performing multiple merging operations on one NFT which is clearly a possible scenario and in the right flow of things.

### Root Cause

Root cause of the issue is that an NFT can practically have an unlimited number of flows

### Internal Pre-conditions

User having NFT from a system related project

### External Pre-conditions

None

### Attack Path

1. User follows the right flow of things and uses the merge functionality to merge his old NFTs with the new ones minted to him
2. User is unable to withdraw his funds because of the number of flows on his NFT

### Impact

Permanent loss of funds for the user 

### PoC

Not needed

### Mitigation

Limit the max number of flows an NFT can support
  