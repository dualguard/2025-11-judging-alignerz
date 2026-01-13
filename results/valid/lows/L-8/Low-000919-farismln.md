# [000919] `AlignerzNFT` pause and unpause does not affect NFT transfers
  
  ### Summary

the pause and unpause function of `AlignerzNFT` suppose to not only pause mint, but also transfer. but currently there are no override in transfer function or the internal function that is correspond with it. making the pause unpause function does not prohibit NFT transfer like it supposed to be.

### Root Cause

[AlignerzNFT.sol#L80C1-L88C6](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/AlignerzNFT.sol#L80C1-L88C6)
```solidity
    /// @notice pauses the contract (minting and transfers)
    function pause() external virtual onlyOwner {
        _pause();
    }

    /// @notice unpauses the contract (minting and transfers)
    function unpause() external virtual onlyOwner {
        _unpause();
    }
```
in the whole contract, there are no override or check when doing `transfer` call

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

the issue is straightforward, owner pausing the contract expecting the mint and transfer to be paused.

### Impact

contract owner unable to pause NFT transfer when its needed

### PoC

_No response_

### Mitigation

add `whenNotPaused` to a overrided transfer function
  