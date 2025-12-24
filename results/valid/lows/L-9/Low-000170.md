# [000170] `getClaimableAmountAndSeconds` will leave dust amounts in the protocol if user claim his tokens in multiple transactions
  
  ### Summary

Imagine the following scenario:
1. User has NFT and he wants to claim rewards by calling [`Vesting::claimTokens()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) function
     - There are 2 possible scenarios which are doing it in 1 transaction (vesting period has passed) and in more than 1 transaction (For example claiming when 40% of the vesting period has passed and then claiming when vesting period has fully passed)
2. If the user pick the second scenario he wont receive the full amount of his flow because of the rounding down nature of `getClaimableAmountAndSeconds` function.

This by Its own doesn't seem like much but assuming it can happen to the majority of the users it may result in pretty significant loss for all the users combined. Not to mention that this round down is unfair and lead to loss of funds for the users who decide to claim their rewards before the full vesting period has elapsed.

### Root Cause

Root cause of this issue is the rounding down nature of `getClaimableAmountAndSeconds` function. it can be seen here:
```solidity
   function getClaimableAmountAndSeconds(
        Allocation memory allocation,
        uint256 flowIndex
    ) public view returns (uint256 claimableAmount, uint256 claimableSeconds) {
        uint256 secondsPassed;
        uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
        uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
        uint256 amount = allocation.amounts[flowIndex];
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }

        claimableSeconds = secondsPassed - claimedSeconds;
 @>       claimableAmount = (amount * claimableSeconds) / vestingPeriod; <@
        require(claimableAmount > 0, No_Claimable_Tokens());
        return (claimableAmount, claimableSeconds);
    }

```

### Internal Pre-conditions

User claiming his rewards in more than 1 tx 

### External Pre-conditions

None

### Attack Path

1. User claim his reward at random point in time and then he claims again when the vesting period has elapsed 
2. Then after the second claim the `getClaimableAmountAndSeconds` function rounds down the claimable amount leading to loss of funds for the user

### Impact

This may result in pretty significant loss for the usr because of the `vestingPeriod` value. For example if it represent 10 months (because it should be divisible by 1 month) and the user claims somewhere in the last month prior to claiming after the period has ended, he may lose up to `2_592_000`(as this is the value of `vestingPeriodDivisor`) units of the reward token.

### PoC

Not needed

### Mitigation

track the claimed amounts and when the vesting period ends subtract it from the whole amount and send the result to the user
  