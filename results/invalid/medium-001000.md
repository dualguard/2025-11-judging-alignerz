# [001000] Users permanently lose their refund rights if they fail to claim before the `claimDeadline`
  
  ### Summary

The protocol enforces a hard `claimDeadline` for refunds. If a user's bid was rejected (or they are entitled to a refund) and they do not call `claimRefund` before the deadline, the `claimRefund` function reverts. Subsequently, the owner can call `withdrawPostDeadlineProfit` to sweep *all* stablecoins from the project, effectively confiscating the user's principal.

### Root Cause

In `AlignerzVesting.sol:886` (`claimRefund`):
```solidity
require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
```
In `AlignerzVesting.sol:956` (`withdrawPostDeadlineProfit`):
```solidity
uint256 amount = biddingProject.totalStablecoinBalance; // Includes unclaimed refunds
biddingProject.stablecoin.safeTransfer(treasury, amount);
`****

### Internal Pre-conditions

1. User has a refundable bid.
2. Deadline passes.

### External Pre-conditions

None

### Attack Path

1. User bids 1000 USDC. Bid is rejected.
2. User forgets to claim refund.
3. Deadline passes.
4. User tries to claim -> Reverts.
5. Owner calls `withdrawPostDeadlineProfit` -> Takes the 1000 USDC.

### Impact

Loss of user funds

### PoC

_No response_

### Mitigation

Allow refunds to be claimed indefinitely. Do not include refundable amounts in the owner's withdrawal calculation (track `refundableBalance` separately).
  