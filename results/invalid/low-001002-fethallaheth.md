# [001002] wrong Indexed reference types in events broke the index filtering
  
  ### Summary

The `TVSsMerged` event declares `uint256[] indexed nftIds`. In Solidity, indexed reference types (like arrays) are hashed, meaning the actual array values are not stored in the topic, only their hash. This makes it impossible for off-chain indexers to filter events involving a specific NFT ID. As per Solidity documentation, a topic can only hold a single word (32 bytes), so reference types are hashed.

### Root Cause


In [AlignerzVesting.sol:1160](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L268):
```solidity
event TVSsMerged(
    // ...
    uint256[] indexed nftIds,
    // ...
);
```
Using `indexed` on an array stores `keccak256(abi.encode(nftIds))` as the topic.


### Internal Pre-conditions

None 

### External Pre-conditions

None

### Attack Path

*

### Impact

broken indexed event makes the offchain fail to fetch data

### PoC

_No response_

### Mitigation

emit separate events for each NFT
  