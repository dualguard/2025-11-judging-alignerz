# [000969] Discrepancy between documentation and refund implementation
  
  ### Summary

When refunding funds to users who did not receive TVS, the betting fee is not refunded, although the documentation promises a “full refund (excluding gas fees)”. The fee is immediately sent to the treasury when the bet is placed and is not taken into account when refunding, which leads to a loss of funds for users.

### Root Cause

The user places a bet and transfers a token in the amount of amount + fee.
```solidity 
 function placeBid(
  ...
    ) external {
// ...code
        biddingProject.stablecoin.safeTransferFrom(
            msg.sender,
            address(this),
            amount
        ); 
        if (bidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(
                msg.sender,
                treasury,
                bidFee
            ); 
// ...code
    }
```

The fee is immediately transferred to the treasury.

Although the documentation provides for a full refund.  
```
4.1.1. Bidding over vesting schedule length: 
...
If a user doesn’t secure an allocation, they can claim their full refund (excluding gas fees).
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The user places a bet. (Pays an additional fee to the protocol.)
2. Does not win TVS and wants to get their funds back.
3. Calls `claimRefund()`

### Impact

Users lose their funds even though they are confident that they will be fully refunded. This undermines trust in the protocol. The documentation misleads users. 



### PoC

_No response_

### Mitigation

The documentation or implementation should be changed to match the main idea of the protocol.
Misleading and unfair to customers. 
  