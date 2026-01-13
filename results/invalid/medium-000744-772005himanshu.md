# [000744] [Medium - 🟠]In Contract `ERC721A.sol` function  `_transfer` missing check for burned `tokenId` Still in circulation loss for Sender, Receiver , Protocol
  
  ### Summary

In Contract `ERC721A.sol` function  `_transfer` missing check for burned `tokenId`. This can lead to transfer of the burned tokenId to other user and there also decrease in balance of the sender `_addressData[from].balance -= 1` and increase in receiver balance `_addressData[to].balance += 1` 

This could lead to many issue like locked  one TokenID in the Contract forever and imaginary ownership of the burned TokenId that is not good for the users of the Protocol . There is should be check that we cannot transfer the `tokenOwnership.burned = true`  cannot be transferred direct revert should be implemented and at  burning of tokenId there is no change in the owner.

```solidity
function _transfer(address from, address to, uint256 tokenId) private {
        TokenOwnership memory prevOwnership = _ownershipOf(tokenId);

        if (prevOwnership.addr != from) revert TransferFromIncorrectOwner();

        bool isApprovedOrOwner =
            (_msgSender() == from || isApprovedForAll(from, _msgSender()) || getApproved(tokenId) == _msgSender());


        // ... skip 


        unchecked {
 @>           _addressData[from].balance -= 1;
 @>          _addressData[to].balance += 1;

            TokenOwnership storage currSlot = _ownerships[tokenId];
  @>          currSlot.addr = to;
            currSlot.startTimestamp = uint64(block.timestamp);

            // If the ownership slot of tokenId+1 is not explicitly set, that means the transfer initiator owns it.
            // Set the slot of tokenId+1 explicitly in storage to maintain correctness for ownerOf(tokenId+1) calls.
            uint256 nextTokenId = tokenId + 1;
            TokenOwnership storage nextSlot = _ownerships[nextTokenId];
            if (nextSlot.addr == address(0)) {
                // This will suffice for checking _exists(nextTokenId),
                // as a burned slot cannot contain the zero address.
                if (nextTokenId != _currentIndex) {
                    nextSlot.addr = from;
                    nextSlot.startTimestamp = prevOwnership.startTimestamp;
                }
            }
        }

        emit Transfer(from, to, tokenId);
        _afterTokenTransfers(from, to, tokenId, 1);
    }
```


```solidity
function _burn(uint256 tokenId, bool approvalCheck) internal virtual {
        TokenOwnership memory prevOwnership = _ownershipOf(tokenId);

        address from = prevOwnership.addr;

        // ... Skip 

        unchecked {
            AddressData storage addressData = _addressData[from];
@>            addressData.balance -= 1;
            addressData.numberBurned += 1;

            // Keep track of who burned the token, and the timestamp of burning.
            TokenOwnership storage currSlot = _ownerships[tokenId];
@>            currSlot.addr = from;
            currSlot.startTimestamp = uint64(block.timestamp);
            currSlot.burned = true; // This correct 

           // .. SKIP 
        }
```

### Root Cause

Missing check in the `_transfer(...) ` for the burned 

```solidity
require(!prevOwnership.burned ,"tokenId is already burned");
```

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

Medium - Loss for the Sender , Receiver , and the Protocol because the burned `tokenId` is Still in circulation in the Protocol In Future owner can take profit from it 

### PoC

_No response_

### Mitigation

```diff
function _transfer(address from, address to, uint256 tokenId) private {
        TokenOwnership memory prevOwnership = _ownershipOf(tokenId);

        if (prevOwnership.addr != from) revert TransferFromIncorrectOwner();

+        require(!prevOwnership.burned ,"tokenId is already burned");

        bool isApprovedOrOwner =
            (_msgSender() == from || isApprovedForAll(from, _msgSender()) || getApproved(tokenId) == _msgSender());

        if (!isApprovedOrOwner) revert TransferCallerNotOwnerNorApproved();
        if (to == address(0)) revert TransferToZeroAddress();

        _beforeTokenTransfers(from, to, tokenId, 1);

        // Clear approvals from the previous owner
        _approve(address(0), tokenId, from);

        // Underflow of the sender's balance is impossible because we check for
        // ownership above and the recipient's balance can't realistically overflow.
        // Counter overflow is incredibly unrealistic as tokenId would have to be 2**256.
        unchecked {
            _addressData[from].balance -= 1;
            _addressData[to].balance += 1;

            TokenOwnership storage currSlot = _ownerships[tokenId];
            currSlot.addr = to;
            currSlot.startTimestamp = uint64(block.timestamp);

            // If the ownership slot of tokenId+1 is not explicitly set, that means the transfer initiator owns it.
            // Set the slot of tokenId+1 explicitly in storage to maintain correctness for ownerOf(tokenId+1) calls.
            uint256 nextTokenId = tokenId + 1;
            TokenOwnership storage nextSlot = _ownerships[nextTokenId];
            if (nextSlot.addr == address(0)) {
                // This will suffice for checking _exists(nextTokenId),
                // as a burned slot cannot contain the zero address.
                if (nextTokenId != _currentIndex) {
                    nextSlot.addr = from;
                    nextSlot.startTimestamp = prevOwnership.startTimestamp;
                }
            }
        }

        emit Transfer(from, to, tokenId);
        _afterTokenTransfers(from, to, tokenId, 1);
    }

```
  