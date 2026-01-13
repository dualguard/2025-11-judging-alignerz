# [000445] Whitelisted users can leverage EIP-7702 to allow non-whitelisted users to participate
  
  ### Summary

The protocol uses a whitelisting mechanism in order to allow only a certain type of actors to bid on a project. As seen with the sponsor, whitelisted users will be set from a list given by the project, and they are supposed to not be trusted.

A malicious whitelisted user can bypass the whitelisting system and set his address as an EIP-7702 smart contract to allow anyone to place a bid on their behalf. This would allow for a wider range of users being able to bid, even with the restriction that there can be only one bid winner from this address.

For a broader read to this type of vulnerability, you can read [this Halborn article](https://www.halborn.com/blog/post/eip-7702-security-considerations).

### Root Cause

Whitelisted users are not checked to not be EIP-7702 when placing their bids.

### Attack Path

Alice is whitelisted for a project. 

She knows there are people who may be interested in bidding in it, so she chooses to make her address a proxy that anyone can call to place or update her bid on her behalf, earning a small fees from them if she wants.

When the project bidding ends, Bob, who used Alice's address to place and win a bid, can claim the NFT for himself.

Together, Alice and Bob effectively bypassed the whitelisting security of the protocol.

### Impact

Users who are not whitelisted are able to participate in the project.

### Mitigation

Here is an example modifier to add to whitelist-protected functions:

```solidity
error DelegationNotAllowed(); 

/// @dev Reject calls that are executing an EIP-7702 delegation stub at msg.sender.
///      EIP-7702 delegation stub prefix: 0xef 0x01 0x00
modifier noEIP7702Delegation() {
    bytes memory code = msg.sender.code;
    if (code.length >= 3) {
        if (code[0] == bytes1(0xef) && code[1] == bytes1(0x01) && code[2] == bytes1(0x00)) {
            revert DelegationNotAllowed();
        }
    }
    _;
}
```

  