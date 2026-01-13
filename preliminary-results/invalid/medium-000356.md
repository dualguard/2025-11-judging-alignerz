# [000356] Wrong check for vesting period in Update bid will force unnecessary Vesting period extension for users who want to add to their bid or create a Denial of Service(DOS).
  
  ### Summary

The `updateBid()` function applies an unconditional vesting-period divisibility check:

```solidity
if (newVestingPeriod > 1) {
    require(
        newVestingPeriod % vestingPeriodDivisor == 0,
        Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
    );
}
```

This check is executed **even when the user is not trying to modify their vesting period**, resulting in unnecessary transaction failures and, in some cases, a complete **Denial of Service (DoS)** preventing users from updating only their bid amount.

Additionally, because `vestingPeriodDivisor` is mutable via `setVestingPeriodDivisor`, a later update by the owner can make a previously valid vesting period invalid during subsequent updates—even if the user does not intend to modify the vesting period.

This forces users to artificially increase their vesting period to satisfy the new divisor, effectively **extending vesting durations**.

### Root Cause


 ```solidity
   /// @notice Updates an existing bid
    /// @param projectId ID of the biddingProject
    /// @param newAmount New amount of stablecoin to commit
    /// @param newVestingPeriod New vesting duration
    function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );

        Bid storage bid = biddingProject.bids[msg.sender];
        uint256 oldAmount = bid.amount;
        require(oldAmount > 0, No_Bid_Found());
        require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());
        require(newVestingPeriod > 0, Zero_Value());
        require(newVestingPeriod >= bid.vestingPeriod, New_Vesting_Period_Cannot_Be_Smaller());

@audit>>        if (newVestingPeriod > 1) {                                                                      // bug check new > old ... medium needs an external condition......
            require(
                newVestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
            );
        }
          uint256 oldVestingPeriod = bid.vestingPeriod;


@audit>>          if (newAmount > oldAmount) {
            uint256 additionalAmount = newAmount - oldAmount;
            biddingProject.totalStablecoinBalance += additionalAmount;
            biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), additionalAmount);
        }


```

The `updateBid` function applies the vesting-period divisibility check **unconditionally**, regardless of whether the user is actually modifying their vesting period.
Specifically, the code checks:

```solidity
if (newVestingPeriod > 1) {
    require(newVestingPeriod % vestingPeriodDivisor == 0, ...);
}
```

This logic is flawed because:

1. **The condition `newVestingPeriod > 1` does not indicate that the user intends to change their vesting period.**
   It only checks that the value is greater than 1, which is always true for nearly all valid vesting periods.

2. **The function does not compare `newVestingPeriod` to the current `bid.vestingPeriod`.**
   Therefore, even if the user is *not modifying the vesting period* and only updating their bid amount, the function still enforces the new divisor rule.

3. **`vestingPeriodDivisor` is mutable.**
   If the owner updates it, a previously valid vesting period may no longer be divisible by the *new* divisor.
   Because the divisibility check is applied even when users do not intend to change vesting, they can no longer call `updateBid()`.


```solidity
   /// @notice Places a bid for token vesting
    /// @param projectId ID of the biddingProject
    /// @param amount Amount of stablecoin to commit
    /// @param vestingPeriod Desired vesting duration
    function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());   //bug bypass by updating .....
        }
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(amount > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );
        require(biddingProject.bids[msg.sender].amount == 0, Bid_Already_Exists());

@audit>>          require(vestingPeriod > 0, Zero_Value());

 @audit>>       require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());

        biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);
        if (bidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, bidFee);
        }
        biddingProject.bids[msg.sender] =
            Bid({amount: amount, vestingPeriod: vestingPeriod});
        biddingProject.totalStablecoinBalance += amount;

        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
    }
```

Divisor is mutable.

```solidity

 /// @notice Changes the vesting period multiples
    /// @dev Only callable by the owner.
    /// @param newVestingPeriodDivisor the new value
    /// @return bool indicating success of the operation.
    function setVestingPeriodDivisor(uint256 newVestingPeriodDivisor) external onlyOwner returns (bool) {
        require(newVestingPeriodDivisor > 0, Zero_Value());

@AUDIT>>        uint256 oldVestingPeriodDivisor = vestingPeriodDivisor;
      
@AUDIT>>    require(
            newVestingPeriodDivisor != vestingPeriodDivisor,
            Same_Value()
        );

@AUDIT>>          vestingPeriodDivisor = newVestingPeriodDivisor;
 
       emit vestingPeriodDivisorUpdated(oldVestingPeriodDivisor, newVestingPeriodDivisor);
        return true;
    }

```


### Internal Pre-conditions

1. Mutable nature of the divisor
2. Check enforcement for vesting period

### External Pre-conditions

_No response_

### Attack Path

1. Bid initiated
2. Change in divisor value
3. Update bid called to increase amount

### Impact

**1. Users cannot update the amount without also satisfying the new vesting rules**
**2. Users are forced into longer vesting periods unnecessarily**


### PoC

_No response_

### Mitigation

Replace:

```solidity
if (newVestingPeriod > 1) {
    require(newVestingPeriod % vestingPeriodDivisor == 0, ...);
}
```

**With:**

```solidity
if (newVestingPeriod > bid.vestingPeriod) {
    // only validate when user attempts to INCREASE vesting period
    require(
        newVestingPeriod % vestingPeriodDivisor == 0,
        Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
    );
}
```

  