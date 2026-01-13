# [000606] Missing Whitelist Enforcement in updateBid
  
  ### Summary

Failing to check if a user is still whitelisted in `updateBid` will cause an unauthorized bid update for whitelisted projects as a removed user will be able to modify their bid, bypassing whitelist restrictions and potentially gaining unfair allocation.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741


When placing a bid it is expected that if the project has whitelist enabled then they user must be whitelisted before they can place the bid.

```javascript
 function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }

        // omitted for simplicity
```

However, The updateBid function allows users to modify their existing bids without verifying whether they are still whitelisted for the project:

```javascript

function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    // ...
    Bid storage bid = biddingProject.bids[msg.sender];
    // No whitelist check here
}

```

If the project has isWhitelistEnabled = true, a user could have been removed from the whitelist after initially placing a bid. Currently, such a user can still increase their bid or extend the vesting period, bypassing intended whitelist restrictions.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path



1. user is whitelisted (initial state).
The project owner includes the user address in the whitelist before or during the bidding window.

2. user places a small initial bid while whitelisted.

3. Owner removes the user for some reason from the whitelist.

4. user calls updateBid(projectId, newAmount, newVestingPeriod).
Because updateBid lacks a whitelist check, the function allows execution even though the attacker is no longer whitelisted.

5. Contract accepts the update and pulls the difference.

6. user increases bid to gain allocation advantage.

### Impact


Unauthorized users may update bids even though they were whitelisted at first and for some reason removed from the whitelist


### PoC

_No response_

### Mitigation

Add a whitelist check at the start of updateBid:

```javascript

if (isWhitelistEnabled[projectId]) {
    require(isWhitelisted[msg.sender][projectId], "User is not whitelisted");
}

```

  