# [000575] Missing Whitelist Verification in `updateBid()` Allows De-whitelisted Users to Increase Bidding Positions and Bypass Access Control
  
  
## Summary

The missing whitelist check in `updateBid()` allows de-whitelisted users to increase their bids in whitelist-enabled projects, bypassing access control and undermining the `removeUsersFromWhitelist()` function.

## Root Cause

In [`AlignerzVesting.sol:742-777`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L776), `updateBid()` does not verify `isWhitelisted[msg.sender][projectId]` when `isWhitelistEnabled[projectId]` is true. Unlike `placeBid()` (lines 710-712), which enforces this check, `updateBid()` only validates project ID, active bidding period, existing bid, and amount/vesting constraints, allowing de-whitelisted users to continue updating their bids.
```js
	// @audit these are the only validations in the function
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );

        Bid storage bid = biddingProject.bids[msg.sender];
        uint256 oldAmount = bid.amount;
        require(oldAmount > 0, No_Bid_Found());
        require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());
        require(newVestingPeriod > 0, Zero_Value());
        require(newVestingPeriod >= bid.vestingPeriod, New_Vesting_Period_Cannot_Be_Smaller());
        // ...
    }
```

## Internal Pre-conditions

1. Owner calls `launchBiddingProject()` with `whitelistStatus` set to `true` to enable whitelisting for a project
2. Owner calls `addUserToWhitelist()` or `addUsersToWhitelist()` to whitelist a user for the project
3. The whitelisted user calls `placeBid()` to create an initial bid
4. Owner calls `removeUserFromWhitelist()` or `removeUsersFromWhitelist()` to remove the user from the whitelist, setting `isWhitelisted[user][projectId]` to `false`
5. The bidding period remains active (`block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed`)

## External Pre-conditions

None

## Attack Path

1. Owner launches a bidding project with whitelisting enabled (`isWhitelistEnabled[projectId] = true`)
2. Owner whitelists User A for the project (`isWhitelisted[userA][projectId] = true`)
3. User A calls `placeBid(projectId, amount1, vestingPeriod1)` to place an initial bid
4. Owner removes User A from the whitelist by calling `removeUserFromWhitelist(userA, projectId)`, setting `isWhitelisted[userA][projectId] = false`
5. User A calls `updateBid(projectId, amount2, vestingPeriod2)` where `amount2 > amount1`, successfully increasing their bid despite no longer being whitelisted
6. User A continues to have the same bidding privileges as whitelisted users, effectively bypassing the whitelist removal

## Impact

The protocol and other bidders are affected because de-whitelisted users can continue increasing their positions, undermining access control. This allows users who should be excluded to maintain or increase their participation, potentially skewing allocation outcomes and reducing fairness for legitimate whitelisted participants.

## Proof of Concept
N/A


## Mitigation

Add the same whitelist check used in `placeBid()` to the beginning of `updateBid()`:

```diff
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
+    if (isWhitelistEnabled[projectId]) {
+        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
+    }
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    // ... rest of the function
}
```
  