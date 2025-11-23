# [000392] Missing Check for Vesting Start Time in the Past
  
  **Summary**
The `getClaimableAmountAndSeconds()` function calculates how many tokens a user can claim based on vesting progress. However, it never validates that the vesting start time has already passed. Instead, it directly computes:

```solidity
if (block.timestamp > vestingPeriod + vestingStartTime) {
    secondsPassed = vestingPeriod;
} else {
    secondsPassed = block.timestamp - vestingStartTime;
}
```

If `block.timestamp < vestingStartTime`, the subtraction produces an underflow condition conceptually, preventing the function from working properly and allowing vesting schedules to be configured incorrectly.

**Recommendation**
The protocol should ensure that vestings cannot be claimed before they begin. Without an explicit check such as:
```solidity
require(block.timestamp >= vestingStartTime, "Vesting has not started");
```
  