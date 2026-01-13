# [000153] Users removed from the whitelist can still update their allocations
  
  The placeBid function checks if a user is whitelisted before they can place a bid. However, the updateBid function is missing this same check.

```solidity
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        //no whitelist check
```

This allows a user who was whitelisted, placed a bid, and was then removed from the whitelist to still be able to modify their existing bid. It bypasses the whitelist restriction for bid updates.
  