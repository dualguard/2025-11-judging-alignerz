# [000294] Missing whitelistCheck in `updateBid` function
  
  
**Summary:** There's a missing whitelist check in the `updateBid` function which allows user who have been removed from the whitelist to be able to update their bid which should't be so

**Root Cause** lack of `isWhitelistEnabled[projectId]` validation check in the the `updateBid` function.  

**Vulnerability Details:** :In the `updateBid` function there's no whitelist check for a `projectId` as this check is only present in the `placeBid` function but after the user has been able to place bid in the `placeBid` function by passing the whitelist check, the user can later be removed from the whitelist by the owner of the contract, yet the user is still able to perform some operation like `updateBid`.
```javascript
 function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        //@audit lack of `isWhitelistEnabled[projectId]` validation check
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );

    ////Rest of the code.....


```

**Impact:**
User will be able to still update their bid even after being removed from the whitelist, and this still gives the user some privilege to perform such operation and it should't be so.



**Tool Used:** Manual review
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741

**POC:**

**Recommended Mitigation** Add a whitelist check as it is in `placeBid` funtion:
```javascript
 function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        //add whitelist check
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );
        
    ////Rest of the code....

```

  