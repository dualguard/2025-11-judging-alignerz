# [000393] Users can claim NFTs before allocations are set
  
  ### Summary

The protocol allows users to successfully call `claimNFT()` before the owner publishes the final allocation Merkle roots via `updateProjectAllocations()`. This results in users minting “empty” or meaningless NFTs that have no associated vesting information.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835C4-L853C6?plain=1

### Root Cause

`claimNFT()` does not check whether:
- Allocations have been published
- Merkle roots have been finalized

`finalizeBids()` resets the root to zero but the contract treats zero as a fully valid root of a trivial tree

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

Users can mint NFTs before off-chain allocation is finalized, undermining the integrity of the auction. This breaks the trust model: NFTs are supposed to represent accepted bids and allocated tokens but users may mint NFTs with arbitrary amounts or stale/unapproved allocation data If off-chain allocation is delayed or fails, NFTs may already exist, leading to:
- Frozen projects
- Incorrect vesting states
- Unrecoverable inconsistencies

### PoC




### Mitigation

- Require allocations to be published before NFT claims:
```solidity
require(biddingProject.allocationsSet, "Allocations not finalized");
```

- Disallow `claimNFT()` when `merkleRoot == 0`
- Reject NFT claims unless project is in claiming state
  