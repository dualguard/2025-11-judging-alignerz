# [000755] `launchBiddingProject` does not validate `startTime` against `block.timestamp`
  
  ### Summary

The bidding project launch function only checks `startTime < endTime` but does not ensure that `startTime` is in the future whose bidding window is already over or never valid, making `placeBid` unusable for that project.

### Root Cause

`launchBiddingProject` accepts arbitrary `startTime` and `endTime` and only enforces `startTime < endTime`, then stores them directly:

```solidity
function launchBiddingProject(
    address tokenAddress,
    address stablecoinAddress,
    uint256 startTime,
    uint256 endTime,
    bytes32 endTimeHash,
    bool whitelistStatus
) external onlyOwner {
    require(tokenAddress != address(0), Zero_Address());
    require(stablecoinAddress != address(0), Zero_Address());
>>  require(startTime < endTime, Starttime_Must_Be_Smaller_Than_Endtime());

    BiddingProject storage biddingProject = biddingProjects[biddingProjectCount];
    biddingProject.token = IERC20(tokenAddress);
    biddingProject.stablecoin = IERC20(stablecoinAddress);
    biddingProject.startTime = startTime;
    biddingProject.endTime = endTime;
    ...
}
```

If `startTime` is already in the past or `endTime <= block.timestamp`, the `placeBid` check

```solidity
require(
    block.timestamp >= biddingProject.startTime &&
    block.timestamp <= biddingProject.endTime &&
    !biddingProject.closed,
    Bidding_Period_Is_Not_Active()
);
```

will always fail, and no user can ever place a bid.

### Impact

A misconfigured project will simply be impossible to bid on.

### Mitigation

In `launchBiddingProject`, add checks such as:

```solidity
require(startTime >= block.timestamp, Starttime_Must_Be_In_The_Future());
require(endTime > startTime, Starttime_Must_Be_Smaller_Than_Endtime());
require(endTime > block.timestamp, Bidding_Period_Is_Not_Active());
```


  