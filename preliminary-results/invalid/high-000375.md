# [000375] H02 Immediate fee transfer in updateBid prevents full refunds
  
  ## Summary

The `AlignerzVesting::updateBid` function immediately transfers the `updateBidFee` to the treasury. This design choice makes it impossible for the contract to fulfill the "full refund" guarantee specified in the whitepaper, as the fee funds are no longer under the contract's control when a refund is requested.

4.1.1 states that:
```
• If a user doesn’t secure an allocation, they can claim their full refund (excluding gas fees).
```

## Root Cause

In `AlignerzVesting::updateBid`, the `updateBidFee` is transferred directly to the treasury:

```solidity
        if (updateBidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(
                msg.sender,
                treasury,
                updateBidFee
            );
        }
```

However, the whitepaper states that users who do not secure an allocation can claim a "full refund". Since the fee is sent to an external treasury, the `claimRefund` function (which only refunds the principal `amount`) cannot return the fee to the user.

## Internal Pre-Conditions

1.  `updateBidFee` must be greater than 0.

## External Pre-Conditions

1.  User updates a bid that is ultimately unsuccessful (rejected).

## Attack Path

1.  User calls `updateBid` with `newAmount` + `updateBidFee`.
2.  `updateBidFee` is sent to Treasury. `additionalAmount` is sent to Contract.
3.  Bid is rejected (e.g., outbid or not selected).
4.  User calls `claimRefund`.
5.  Contract refunds `amount`.
6.  User loses `updateBidFee`, violating the "full refund" promise.

## Impact

**High**. Violation of protocol specification and loss of funds for users. Users are promised a full refund but lose the bid update fee.

## PoC

Trivial. Inspect `updateBid` (fee to treasury) and `claimRefund` (only refunds principal).

## Mitigation

**Option A (Recommended):** Hold the `updateBidFee` in the `AlignerzVesting` contract until the bid is finalized.
*   If accepted: Transfer fee to treasury.
*   If rejected: Refund `amount + fees` to user.

**Option B:** Update the whitepaper to explicitly state that bid fees are non-refundable.
  