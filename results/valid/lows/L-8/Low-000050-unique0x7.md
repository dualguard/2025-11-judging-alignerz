# [000050] `Pause` and `unpause` functions do not `pause` and `unpause` transfers
  
  
**Note:** This issue is different from the `LightChaser` bot finding which is 
>  [Medium-4]. The contract controlling a Pausable contract fails to expose its `pause`/`unpause` functions, even though they are defined.

### Summary

The `AlignerzNFT` uses OpenZeppelin’s `Pausable` module and applies `whenNotPaused` to minting and burning functions. However, standard ERC721 transfers (`transferFrom`, `safeTransferFrom`) remain completely unrestricted. This means that calling `pause()` does not stop token transfers, even though the contract owner would reasonably expect all movement of tokens to be halted during emergency conditions.


### Root Cause

```solidity
    /// @notice pauses the contract (minting and transfers) //@audit   transfer is not paused
    function pause() external virtual onlyOwner {
        _pause();
    }

    /// @notice unpauses the contract (minting and transfers) ////@audit   transfer is not paused
    function unpause() external virtual onlyOwner {
        _unpause();
    }

```

[link to Pause()](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol#L80) , [link to unpause()](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol#L86)


### Attack Path

Based on these comments, minting and transfers are supposed to be paused. However, `transferFrom` and `safeTransferFrom` are not actually restricted ERC721A does not automatically block transfers.

```solidity
    /// @notice pauses the contract (minting and transfers) //@audit   transfer is not paused
    function pause() external virtual onlyOwner {
     ...

    /// @notice unpauses the contract (minting and transfers) ////@audit   transfer is not paused
    function unpause() external virtual onlyOwner {
    ...

```

### Impact

- Unwanted token movement during an emergency
 
- Tokens being moved while the owner believes the system is in a secure “frozen” state


### Mitigation

Add the `whenNotPaused` modifier to all transferable functions to ensure transfers are blocked when the contract is paused.
  