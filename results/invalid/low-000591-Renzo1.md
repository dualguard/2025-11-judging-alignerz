# [000591] Missing Bid Cancellation in Whitelist Removal Causes Invalid Bids to Remain Active and Undermines Access Control
  
  
&nbsp;

## Summary

The lack of bid cancellation logic in `removeUsersFromWhitelist` will cause invalid bids to remain active for the protocol and removed users, as an admin removing a user from the whitelist will not invalidate or refund their existing bid, allowing non-whitelisted users to participate in the bidding process.

&nbsp;

## Root Cause

In [`WhitelistManager.sol:140-144`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L140-L145), the `_removeUserFromWhitelist` function only sets `isWhitelisted[user][projectId] = false` but does not cancel or refund existing bids stored in `biddingProjects[projectId].bids[user]`. Additionally, there is no general function to cancel user bids in the contract, meaning once a bid is placed, it can only be updated, refunded after project closure (via merkle proof), or claimed as an NFT if accepted.

```js
    function _removeUserFromWhitelist(address user, uint256 projectId) internal {
        require(user != address(0), "Invalid address");
        require(isWhitelisted[user][projectId], "address is not whitelisted");
        isWhitelisted[user][projectId] = false;
        emit userBlacklisted(projectId, user);
    }
```
&nbsp;

## Internal Pre-conditions

1. Admin needs to call `launchBiddingProject()` to set `isWhitelistEnabled[projectId]` to `true` for a bidding project
2. Admin needs to call `addUsersToWhitelist()` to set `isWhitelisted[user][projectId]` to `true` for at least one user
3. User needs to call `placeBid()` to create a bid entry in `biddingProjects[projectId].bids[user]` with `amount > 0`
4. Bidding period needs to be active (block.timestamp >= startTime && block.timestamp <= endTime && !closed)
5. Admin needs to call `removeUsersFromWhitelist()` to set `isWhitelisted[user][projectId]` to `false` while the user's bid remains in `biddingProjects[projectId].bids[user]`

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. Admin launches a bidding project with whitelisting enabled via `launchBiddingProject()` with `whitelistStatus = true`
2. Admin adds User A to the whitelist via `addUsersToWhitelist()` for the project
3. User A places a bid via `placeBid()` with a stablecoin amount, creating a bid entry in `biddingProjects[projectId].bids[userA]`
4. Admin removes User A from the whitelist via `removeUsersFromWhitelist()`, setting `isWhitelisted[userA][projectId] = false`
5. User A's bid remains active in `biddingProjects[projectId].bids[userA]` with `amount > 0`
6. When the project closes, User A can still claim an NFT via `claimNFT()` if included in the merkle tree, or claim a refund via `claimRefund()` if included in the refund merkle tree, despite no longer being whitelisted

&nbsp;

## Impact

The protocol suffers a loss of access control integrity as removed users can still participate in the bidding outcome. Removed users may receive token allocations or refunds they should not be eligible for, undermining the whitelist mechanism. The protocol cannot effectively revoke participation rights once a bid is placed, creating a permanent access control bypass for users who were whitelisted at bid time but later removed.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Add bid cancellation logic to `_removeUserFromWhitelist` in `WhitelistManager.sol` that:
1. Checks if the user has an active bid for the project
2. If a bid exists and the project is still open, deletes the bid entry and refunds the stablecoin amount
3. Updates `biddingProject.totalStablecoinBalance` accordingly

Alternatively, implement a separate `cancelBid()` function that allows the owner to cancel specific user bids, which can be called when removing users from the whitelist. 


  