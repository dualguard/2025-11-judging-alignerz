# [000601] Potential Unexpected Execution Flow Hijack Through Reentrancy
  
  ### Summary

In [`AlignerzVesting.sol` lines 860–891](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-L891) and [`AlignerzVesting.sol` lines 1054–1107](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107), the contract mints an NFT **before** updating the allocation state. When the NFT receiver is a smart contract, `_safeMint` triggers a callback (`onERC721Received`) on the receiver. This allows the receiver contract to reenter the vesting contract while the state is still outdated, creating a potential reentrancy vector that can lead to unexpected execution flow hijacking.


### Root Cause

**Root Cause:**
The contract performs `_safeMint` **before** updating the allocation state in both locations:

* [`AlignerzVesting.sol` lines 860–891](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-L891)
* [`AlignerzVesting.sol` lines 1054–1107](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107)

Because `_safeMint` invokes `onERC721Received` when the receiver is a contract, this callback is triggered **before** the vesting state is updated. As a result, the external contract can reenter the vesting contract while it is in an inconsistent or outdated state, enabling an unexpected execution flow hijack through reentrancy.


### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

1. The attacker calls `claimNFT` or performs a TVS split, which triggers an NFT mint.
2. During the callback invoked by `_safeMint`, the attacker can reenter the contract, creating a potential execution flow hijack.


### Impact

Unexpected Execution Flow

### PoC

-

### Mitigation

consider adding reentrancy protection like nonReentrant modifier
  