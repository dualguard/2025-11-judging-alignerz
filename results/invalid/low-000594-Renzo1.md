# [000594] Missing Post-Deadline Claim Mechanism and Refund Recovery in Bidding Projects Causes Permanent Loss of User Funds and NFTs
  
  ## Summary

The lack of a post-deadline claim mechanism and automatic refund recovery in bidding projects will cause a complete loss of funds and NFTs for users who fail to claim within the deadline, as the protocol owner will withdraw all unclaimed stablecoin balances to the treasury after the claim deadline passes, and users have no way to claim their NFTs or receive refunds after the deadline.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:866`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L865) and [`AlignerzVesting.sol:838`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L837), the `claimNFT()` and `claimRefund()` functions enforce a strict deadline check that prevents users from claiming after `biddingProject.claimDeadline` has passed. Unlike reward projects which have a `distributeRemainingRewardTVS()` function at `AlignerzVesting.sol:554` that allows the owner to distribute unclaimed NFTs after the deadline, bidding projects have no equivalent recovery mechanism. Additionally, in `AlignerzVesting.sol:804`, the `claimDeadline` is only set when the owner calls `finalizeBids()`, meaning users cannot know the claim window duration until after the bidding period has closed. After the deadline passes, the owner can withdraw all remaining stablecoin balances via `_withdrawPostDeadlineProfit()` at `AlignerzVesting.sol:927-935`, transferring unclaimed funds to the treasury without providing users any recourse to claim their NFTs or receive refunds.

&nbsp;

## Internal Pre-conditions

1. Owner needs to call `finalizeBids()` to set `claimDeadline` to `block.timestamp + claimWindow` where `claimWindow` is chosen by the owner
2. `block.timestamp` needs to exceed `biddingProject.claimDeadline` for the deadline to pass
3. At least one user needs to have placed a bid but failed to call `claimNFT()` or `claimRefund()` before the deadline
4. Owner needs to call `withdrawPostDeadlineProfit()` or `withdrawAllPostDeadlineProfits()` after the deadline has passed

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User places a bid by calling `placeBid()` with stablecoin tokens, committing funds to the bidding project
2. Owner calls `finalizeBids()` with a `claimWindow` parameter, setting the `claimDeadline` to `block.timestamp + claimWindow` and closing the bidding period
3. User is unaware of the claim deadline duration until after the bid is finalized, as the deadline is only revealed at finalization
4. `block.timestamp` exceeds `biddingProject.claimDeadline`, making the deadline check in `claimNFT()` and `claimRefund()` fail
5. User attempts to call `claimNFT()` or `claimRefund()` but the transaction reverts due to the `Deadline_Has_Passed()` requirement at lines 866 and 838
6. Owner calls `withdrawPostDeadlineProfit()` which transfers all remaining `totalStablecoinBalance` to the treasury, permanently removing user funds from the contract

&nbsp;

## Impact

Users who fail to claim their NFTs or refunds before the deadline suffer a complete loss of their committed stablecoin funds. The protocol owner gains these funds through the treasury withdrawal, while affected users receive neither their NFTs nor refunds.


&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Implement a post-deadline recovery mechanism similar to reward projects.
  