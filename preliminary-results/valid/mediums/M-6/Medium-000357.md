# [000357] Unwhitelisted users can still add to their bids
  
  ### Summary

The update bid function lacks a whitelisting check that allows a user who has been unwhitelisted before the bid ends to still add to their bid.

### Root Cause

```solidity
    /// @notice Places a bid for token vesting
    /// @param projectId ID of the biddingProject
    /// @param amount Amount of stablecoin to commit
    /// @param vestingPeriod Desired vesting duration
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {

@audit>>        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());   //bug bypass by updating .....
        }
 
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(amount > 0, Zero_Value());
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L708-L711

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L737-L747

The whitelisting model is flexible, and the whitelisting role can be granted and revoked at any point 


```solidity

@audit>>  /// @notice Adds a user to the whitelist (internal logic)
    /// @param user address of the user to add to the whitelist
    /// @param projectId ID of the project
    function _addUserToWhitelist(address user, uint256 projectId) internal {
        require(user != address(0), "Invalid address");
        require(!isWhitelisted[user][projectId], "Already whitelisted");
        isWhitelisted[user][projectId] = true;
        emit userWhitelisted(projectId, user);
    }


@audit>>     /// @notice Removes users from the whitelist
    /// @param users list of the addresses of the users to remove from the whitelist
    /// @param projectId ID of the project
    function removeUsersFromWhitelist(address[] calldata users, uint256 projectId) external onlyOwner {
        uint256 length = users.length;
        for (uint256 i = 0; i < length;) {
            _removeUserFromWhitelist(users[i], projectId);
            unchecked {
                ++i;
            }
        }
    }
```




But the update function still fails to uphold this

```solditiy
  /// @notice Updates an existing bid
    /// @param projectId ID of the biddingProject
    /// @param newAmount New amount of stablecoin to commit
    /// @param newVestingPeriod New vesting duration
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {

@auditt>> // no whitelisting enforcement.

        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );
```

Also, project whitelisting status can also be updated because they are flexible.

### Internal Pre-conditions

1. Flexible whitelisting model for both projects and users 

### External Pre-conditions

_No response_

### Attack Path

1. Initial Bid 
2. Loss of whitelisting role
3. The user can still add to their bid (bid for more).


Another path is

1. Initial Bid
2. Project switches enabling status to true
3. Unwhitelisted User can still add to their bid (bid for more).

### Impact

Bypassing the whitelisting model allows users without the role to add to their bid.

### PoC

_No response_

### Mitigation

Add the whitelisting check to the Update bid 
  