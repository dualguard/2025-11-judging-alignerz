# [001008] Whitelisted addresses may not get an allocation because `updateBid()` doesn't check whitelist status
  
  ### Summary
`AlignerzVesting` contract implements an user whitelist mechanism which can be enabled per projectId.
While [placeBid()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L708-L711) check if the user placing the bid is whitelisted, the same check is missing from [updateBid()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L776) function. 
```solidity
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
//...
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        // @audit-low info updating the bid doesn't check the whitelist status
        require(projectId < biddingProjectCount, Invalid_Project_Id());
//...
```

Problems can appear if the following scenario: 
1. admin launches a new bidding project with no whitelist
2. after a while admin decide to enable the whitelist to ensure from that moment just allowed addresses can bid. 
The already placed bids are considered valid and will not be excluded from pools allocation. 
3. Users who have active bids but are not whitelisted edit their bids by calling `updateBid()`: they can increase the `vestingPeriod` or bid `amount` and whitelisted addresses will be excluded from the pools allocation. 

### Impact
Allow listed addresses may not get an allocation when the whitelist is activated  after bids have started.

### Mitigation
Include the following check in `updateBid()` function :
```diff
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
+        if (isWhitelistEnabled[projectId]) {
+            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
+        }
//...
```
  