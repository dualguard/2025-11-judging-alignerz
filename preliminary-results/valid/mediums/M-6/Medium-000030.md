# [000030] Removed whitelisted users can continue updating their bids
  
  ### Summary

When placing a bid, the contract correctly enforces the project whitelist if `isWhitelistEnabled[projectId]` is true:

```solidity
function placeBid(
    uint256 projectId,
    uint256 amount,
    uint256 vestingPeriod
) external {
    if (isWhitelistEnabled[projectId]) {
        require(
            isWhitelisted[msg.sender][projectId],
            User_Is_Not_whitelisted()
        );
    }
    // ...
}
```

However, `updateBid` does **not** perform the same whitelist check:

```solidity
function updateBid(
    uint256 projectId,
    uint256 newAmount,
    uint256 newVestingPeriod
) external {
    require(projectId < biddingProjectCount, Invalid_Project_Id());
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(
        block.timestamp >= biddingProject.startTime &&
            block.timestamp <= biddingProject.endTime &&
            !biddingProject.closed,
        Bidding_Period_Is_Not_Active()
    );

    Bid storage bid = biddingProject.bids[msg.sender];
    uint256 oldAmount = bid.amount;
    require(oldAmount > 0, No_Bid_Found());
    // ...
}
```

This means:

* A user can be whitelisted at the time of `placeBid` and successfully open a position.
* Later, the project can remove that user from the whitelist.
* Despite that, the user can still call `updateBid` and change their bid (increase amount / extend vesting) as long as the bidding period is open.

Whitelist status therefore only gates **initial bid creation**, not **subsequent bid updates**.


### Root Cause

Not checking if the bidder is whitelisted when updating a bid, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L741.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Users can still update their bids even if they were removed from the whitelist.

### PoC

_No response_

### Mitigation

Consider mirroring the whitelist enforcement from `placeBid` into `updateBid` whenever the project has whitelisting enabled.
  