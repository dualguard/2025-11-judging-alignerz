# [000176] Flawed logic in `claimDividends` function might lead to impossibility to claim rewards for users.
  
  ### Summary

As explained by the developer of the project, the A26ZDividendDistributor contract is responsible for distributing dividends to TVS holders (TVS for A26Z tokens, but potentially others tokens in the future as the token can be changed in the contract).
Initially, the vesting period will be 3 months. There will be a new round of dividends distribution every 3 months.

`claimDividends` is a function that users call in order to receive their dividend for a round of distribution. It is defined as follows:

```solidity
    function claimDividends() external {
        address user = msg.sender;
        uint256 totalAmount = dividendsOf[user].amount;
        uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
        uint256 secondsPassed;
        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }
        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
        emit dividendsClaimed(user, claimableAmount);
    }
```

We can see that it retrieves `dividendsOf[user].amount;` and `dividendsOf[user].claimedSeconds` and computes the seconds passed since the start time and the claimable seconds. It updates `dividendsOf[user].claimedSeconds`, computes the claimable amount and transfers it to the user.

The issue arises because this design works as long as there is only one distribution round. Once the owner resets `startTime` , things will start to mess up. By the way, the `setUpTheDividends` function doesn't include an update of the `startTime` while it should ideally. With the current design, the owner needs to call both `setUpTheDividends` and `setStartTime`.

Especially, the real issue is that the line:
```solidity
dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
```
might underflow, leading to impossibility for users to claim their dividends.

### Root Cause

The root cause lies in a specific path. Overall, there are 2 possibilities:
- a user claims his dividends entirely before the new round is set up and the `if` branch is executed: `dividendsOf[user].amount` and `dividendsOf[user].claimedSeconds` are reset to 0. In that case, there is no issue
- a user only claims partially his dividends before the next round is set up. In that case, `dividendsOf[user].claimedSeconds`
 is not reset.   This means that if the user calls `claimDividends` right after the beginning of the next round, the line `dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);` will underflow because `claimedSeconds` will be bigger than `secondsPassed`

### Attack Path

There is no specific attack path. The `claimDividends` function is globally problematic. If only one round of distribution happened, it could be ok. But there will be loss of rewards for users who don't claim entirely their allocation for a given round. 

### Impact

The impact of this issue is high as it leads to loss of dividends for users.

### Mitigation

The dividend distribution feature needs to be reworked globally so that it works flawlessly and distributes dividends every 3 months, with users allowed to claim whenever they want.
  