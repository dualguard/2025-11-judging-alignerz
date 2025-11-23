# [000348] `ERC721A.sol::_ownershipOf#180` may underflow during backward ownership scan allowing unexpected revert
  
  ### Summary

The absence a boundry check in the backward scanning logic of `_ownershipOf` will cause an underflow, resulting in an unexpected revert for token ownership queries. This affects the protocol users as it reverts when `_ERC721A::ownershipOf` tries to scan below `ERC721A::_startTokenId()`.


### Root Cause

In [`ERC721A.sol::_ownershipOf#180`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L180): 

```solidity
    function _ownershipOf(uint256 tokenId) internal view returns (TokenOwnership memory) {
        // ...

        unchecked {
            if (...) {
                    // ...
                    while (true) {
                        curr--;
                        ownership = _ownerships[curr];
                        if (ownership.addr != address(0)) {
                            return ownership;
                        }
                    }
                }
            }
        }
        // ...
    }
```
This code performs `curr--` inside a `while(true)` loop without verifying that `curr > _startTokenId()`.
This allows `curr` variable to underflow from `_startTokenId()`, breaking the ERC721A invariants.

### Internal Pre-conditions

1. The NFT collection is using ERC721A style here.
2. `_ownershipOf(tokenId)` is called for a token

### External Pre-conditions

_No response_

### Attack Path

1. User calls a function that relies on `_ownershipOf(tokenId)` like in `_transfer`, `owner`, and `extOwnerOf` function.
2. If `_ownershipOf` checks the slot and does not find explicit ownership.
3. The code enters the backward scan:
   ```solidity
   curr--;
   ```
4. If `curr == _startTokenId()`, decrementing  makes this underflow to `2**256 - 1`.

### Impact

All platform users are affected.
NFT operations using `_ownershipOf` may revert unexpectedly, breaking the protocols core functionality.

### PoC

_No response_

### Mitigation

We should add a lower bound check before decrementing:

```solidity
    while (curr > _startTokenId()) {
        curr--;
        ownership = _ownerships[curr];
        if (ownership.addr != address(0)) {
            return ownership;
        }
    }
```

We can also follow the official ERC721A logic implementation:
```solidity
    while (true) {
        if (curr == _startTokenId()) revert();
        curr--;
    }
```
  