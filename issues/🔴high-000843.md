# [000843] Whitelist Bypass Allows Blacklisted Users to Continue Bidding
  
  ## Summary

The `updateBid` function lacks whitelist validation, allowing users who have been intentionally removed from the whitelist ("blacklisted") to continue increasing their bids, violating admin intent and defeating the restrictive purpose of whitelist removal.

## Vulnerability Detail

The protocol implements a whitelisting mechanism where removing users from the whitelist is explicitly treated as "blacklisting" them. However, the `updateBid` function fails to enforce whitelist validation, allowing blacklisted users to continue escalating their bid participation.

**Admin Intent Evidence**: The protocol clearly indicates removing users is intended as a punitive/restrictive measure:

**Location**: [`WhitelistManager.sol:137-144`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L137-L144)

```solidity
/// @notice Removes a user from the whitelist (internal logic)
function _removeUserFromWhitelist(address user, uint256 projectId) internal {
    require(user != address(0), "Invalid address");
    require(isWhitelisted[user][projectId], "address is not whitelisted");
    isWhitelisted[user][projectId] = false;
    emit userBlacklisted(projectId, user); // ← KEY: "blacklisted" not "removed"
}
```

**The `userBlacklisted` event name indicates admin intent to RESTRICT further participation.**

### Access Control Inconsistency

**`placeBid()` - PROPERLY ENFORCES WHITELIST**:

**Location**: [`AlignerzVesting.sol:708-711`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L708-L711)

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    if (isWhitelistEnabled[projectId]) {
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted()); // ✓ PROTECTED
    }
    // ... rest of function
}
```

**`updateBid()` - VIOLATES BLACKLIST INTENT**:

**Location**: [`AlignerzVesting.sol:741-755`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L755)

```solidity
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    // ❌ MISSING: Whitelist validation to respect "blacklist" intent
    
    Bid storage bid = biddingProject.bids[msg.sender];
    require(oldAmount > 0, No_Bid_Found()); // Only checks if bid exists
    // Blacklisted users can still increase their bids!
}
```

### Root Cause Analysis

**Intent Violation**: Users can only call `updateBid()` if they have an existing bid (requiring prior whitelist access), but once removed from whitelist ("blacklisted"), they can still increase their participation level, defeating the restrictive purpose of blacklisting.

## Impact

Access control bypass enabling continued participation despite restrictions

Blacklisted users can continue increasing participation despite removal
Defeats the restrictive purpose of whitelist removal

## Code Snippet

```solidity
// AlignerzVesting.sol:708-711 - Proper whitelist enforcement
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    if (isWhitelistEnabled[projectId]) {
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted()); // ✓ ENFORCED
    }
    // ... bid placement logic
}

// AlignerzVesting.sol:741-755 - Missing whitelist enforcement
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(
        block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
        Bidding_Period_Is_Not_Active()
    );

    Bid storage bid = biddingProject.bids[msg.sender];
    uint256 oldAmount = bid.amount;
    require(oldAmount > 0, No_Bid_Found()); // Only checks bid existence
    require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());
    // ❌ MISSING: Whitelist check to respect admin blacklist intent
}
```


## Recommendation

**1. Add whitelist validation to updateBid function**:

```solidity
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    // FIX: Add whitelist check at the beginning
    if (isWhitelistEnabled[projectId]) {
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
    }
    
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
    // ... rest of function logic
}
```


This access control bypass allows circumvention of admin restrictions and violates the intended security model where whitelist removal is meant to restrict user participation.
  