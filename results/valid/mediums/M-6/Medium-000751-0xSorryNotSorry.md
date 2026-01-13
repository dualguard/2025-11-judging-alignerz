# [000751] `updateBid` ignores whitelist, so delisted users can still modify existing bids after whitelisting is enabled
  
  
### Summary

The `updateBid` function does not enforce the per-project whitelist, so a user who was initially allowed to bid but later removed from the whitelist (or added after whitelisting is turned on) can still update their existing bid even when new bids are blocked by the whitelist.

### Root Cause

`placeBid` checks the whitelist when `isWhitelistEnabled[projectId]` is true, but `updateBid` does not:

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    if (isWhitelistEnabled[projectId]) { 
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
    }
    ...
}
```

```solidity
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external { 
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(
        block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
        Bidding_Period_Is_Not_Active()
    );
    ...
}
```

If the owner enables whitelisting after some bids exist or removes an address from the whitelist mid-auction, that address cannot call `placeBid` again but can still call `updateBid` and increase their amount or change vesting terms.

### Impact

Previously accepted bidders are still allowed to modify their bid even if whitelisting is turned on or they get removed later, which weakens the strictness of whitelisting but does not directly enable theft or bypass of the initial admission gate.

### Mitigation

Mirror the whitelist check from `placeBid` inside `updateBid`:

```solidity
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    if (isWhitelistEnabled[projectId]) {
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
    }
    ...
}
```
  