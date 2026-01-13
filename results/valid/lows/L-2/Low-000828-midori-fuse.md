# [000828]  `withdrawAllPostDeadlineProfits()` will withdraw all profits including pre-deadlines even though it should only withdraw post-deadlines
  
  ### Summary

The `claimDeadline` defaults to 0 for new bidding projects, allowing owner to withdraw bidder funds from unfinalized projects since `block.timestamp > 0` always passes.

This also causes `withdrawAllPostDeadlineProfits()` to withdraw **all** profits (including pre-deadline) without the admin knowing or being able to do anything about it


### Root Cause


In [AlignerzVesting.sol:812-829](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L812-L829), `launchBiddingProject()` does not initialize `claimDeadline`:

```solidity
function launchBiddingProject(...) external onlyOwner {
    // ...
    BiddingProject storage biddingProject = biddingProjects[biddingProjectCount];
    biddingProject.token = IERC20(tokenAddress);
    biddingProject.stablecoin = IERC20(stablecoinAddress);
    biddingProject.startTime = startTime;
    biddingProject.endTime = endTime;
    biddingProject.poolCount = 0;
    biddingProject.endTimeHash = endTimeHash;
    // claimDeadline remains 0 (default value)
    // ...
}
```

The `claimDeadline` is only set later in [AlignerzVesting.sol:803](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L803) when `finalizeBids()` is called:

```solidity
function finalizeBids(...) external onlyOwner {
    // ...
    biddingProject.claimDeadline = block.timestamp + claimWindow;
    // ...
}
```

However, the owner can call `withdrawPostDeadlineProfit()` in [AlignerzVesting.sol:914-922](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L914C5-L922C6) before finalization:

```solidity
function withdrawPostDeadlineProfit(uint256 projectId) external onlyOwner {
    //...
    uint256 deadline = biddingProject.claimDeadline; // deadline = 0
    require(block.timestamp > deadline, Deadline_Has_Not_Passed()); // Always passes since block.timestamp > 0
    //...
}
```

Since `claimDeadline` defaults to 0, the check `block.timestamp > deadline` always passes, allowing the owner to withdraw all bidder funds at any time before finalization.

The same issue exists in [AlignerzVesting.sol:903-910](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L903-L910), [AlignerzVesting.sol:926-935](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L926-L935):

```solidity
function withdrawAllPostDeadlineProfits() external onlyOwner {
    uint256 _projectCount = biddingProjectCount;
    for (uint256 i; i < _projectCount; i++) {
        _withdrawPostDeadlineProfit(i); // Withdraws from all projects, including unfinaliz ones
    }
}
```

```solidity
function _withdrawPostDeadlineProfit(uint256 projectId) internal {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    uint256 deadline = biddingProject.claimDeadline;
    if (block.timestamp > deadline) {
        uint256 amount = biddingProject.totalStablecoinBalance;
        biddingProject.stablecoin.safeTransfer(treasury, amount);
        biddingProject.totalStablecoinBalance = 0;
        //...
    }
}
```

This function allows the owner to drain all bidding projects in a single transaction, including those that haven't been finalized yet.

### Internal Pre-conditions

1. At least one bidding project has not been finalized yet
2. Owner calls `withdrawPostDeadlineProfit()` on an active bidding project or calls `withdrawAllPostDeadlineProfits()`


### External Pre-conditions

None

### Attack Path


1. Owner launches two bidding projects
2. Users place bids in both projects:
   - **Project 0**: 500,000 USDC
   - **Project 1**: 1,000,000 USDC
3. Owner finalizes Project 0 with `finalizeBids()`
4. Project 0 claim period expires, making it legitimate to withdraw profits
5. Owner calls `withdrawAllPostDeadlineProfits()` to withdraw profits from Project 0
6. The function loops through all projects (0 and 1):
   - **Project 0**: `claimDeadline` has passed, 500,000 USDC withdrawn (legitimate)
   - **Project 1**: `claimDeadline = 0`, **check `block.timestamp > 0` passes, 1,000,000 USDC withdrawn**
7. Both projects are drained in a single transaction. Project 1 users lose all their 1,000,000 USDC even though the project is still active and not finalized

### Impact

- The owner can call `withdrawAllPostDeadlineProfits()`, thinking they would get all closed bidding project profits in 1 transaction, but will unknowingly also drain active bidding projects (primary impact).
- The owner can steal all bidder funds from any active bidding project before finalization (secondary impact).


### PoC

_No response_

### Mitigation


Initialize `claimDeadline` to `type(uint256).max` during project launch so the withdrawal check fails until `finalizeBids()` sets the proper deadline:

```diff
function launchBiddingProject(...) external onlyOwner {
    // ...
+   biddingProject.claimDeadline = type(uint256).max;
    // ...
}
```

  