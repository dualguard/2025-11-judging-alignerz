# [000098] Missing checks for whitelisted users allow unwhitelisted user to still interact with protocol functionalities
  
  ### Summary

Function [`placeBid`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L708) has a check for whitelisted users at the beginning:
```solidity
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
@>            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
        ...
```
But functions like [`updateBid`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741), [`claimRefund`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835) and [`claimNFT`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860) do not have this check. This allows users that were previously whitelisted and called `placeBid` and that got unwhitelisted before bidding project finalized to still call other functions.

### Root Cause

No whitelisting checks in functions [updateBid](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741), [claimRefund](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835) and [claimNFT](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860).

### Internal Pre-conditions

1. User was whitelisted and has gotten unwhitelisted after calling `placeBid`.

### External Pre-conditions

None

### Attack Path

1. User gets whitelisted.
2. User calls `placeBid`.
3. User gets unwhitelisted.
4. User can still `updateBid` or claim rewards or refunds.

### Impact

A user that should not be able to interact with a bidding project and that is unwhitelisted, can still do it if when he entered the bid he was whitelisted.

### PoC

None

### Mitigation

Add checks at the start of the functions.
  