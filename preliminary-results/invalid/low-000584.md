# [000584] Missing `from != to` validation in `ERC721A::_transfer` allows self-transfers that reset ownership timestamp and corrupt time-sensitive state for integrated protocols
  
  

## Summary

The lack of validation preventing transfers where `from == to` in `ERC721A::_transfer` will cause incorrect time-sensitive state reads for protocols integrating Alignerz NFTs, as an attacker can transfer tokens to themselves to reset the `startTimestamp` to the current block timestamp, corrupting ownership duration calculations.

## Root Cause

In [`ERC721A.sol:406-431`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L406-L432), the `_transfer` function lacks a check to prevent self-transfers (`from == to`). On line 431, `currSlot.startTimestamp` is unconditionally set to `uint64(block.timestamp)` during any transfer, including when `from == to`. This resets the ownership timestamp even though ownership does not change, corrupting the stored ownership start time.

```js
    function _transfer(...) private {
        // ...
            currSlot.startTimestamp = uint64(block.timestamp);
		// ...
}
```

## Internal Pre-conditions

1. A user must own at least one NFT token


## External Pre-conditions

None required

## Attack Path

1. Attacker owns token `tokenId` with `startTimestamp = T1`
2. Attacker calls `transferFrom(attacker, attacker, tokenId)` or `safeTransferFrom(attacker, attacker, tokenId)`
3. `_transfer` executes without a `from != to` check
4. `currSlot.startTimestamp` is updated to `block.timestamp` (line 431), resetting the timestamp to `T2` where `T2 > T1`
5. The ownership timestamp is now incorrect, showing a later start time than the actual ownership start

## Impact

Protocols integrating Alignerz NFTs that rely on `startTimestamp` for time-based logic (e.g., ownership duration, rewards, vesting) will read incorrect data. While no direct loss is identified in the Alignerz protocol, this can be exploited in integrated protocols where time-sensitive data is critical, potentially affecting reward calculations, eligibility checks, or other time-dependent operations.

## Proof of Concept
N/A

## Mitigation

Add a check at the beginning of `_transfer` to prevent self-transfers:

```diff
function _transfer(address from, address to, uint256 tokenId) private {
    // ...
    
    // @audit Add this check
+    if (from == to) revert TransferToSameAddress();

    // ...
}
```

  