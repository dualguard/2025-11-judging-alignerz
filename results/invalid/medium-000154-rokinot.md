# [000154] Token claiming has incorrect timestamp verification, leading to DoS before vesting is finished
  
  The function getClaimableAmountAndSeconds calculates the elapsed time of a vesting schedule with the lines below:

```solidity
       if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }
```

If claimTokens is called before this future vestingStartTime is reached, the subtraction will result in an underflow causing the entire transaction to revert. For vesting schedules that are set to start in the future, any attempt by a user to claim tokens from their NFT before that start time will fail with a reverted transaction, blocking them from claiming until the end of the vesting.

To fix it, add a check within getClaimableAmountAndSeconds. Before performing the subtraction, verify if the vesting period has started. If block.timestamp < vestingStartTime, the function should return 0 for both claimableAmount and claimableSeconds instead of attempting the calculation that would underflow.
  