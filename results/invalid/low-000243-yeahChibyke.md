# [000243] Underflow Protection Provides Unclear Error When Users Claim Before Vesting Start Time
  
  - **Description**

The `A26ZDividendDistributor::claimDividends()` function calculates `secondsPassed = block.timestamp - startTime` without explicitly checking if `block.timestamp >= startTime`. When users attempt to claim before the vesting start time, the subtraction underflows and reverts, but the error message is not one a user can understand.

- **Impact**

- Poor user experience

- **Recommended Mitigation**

Add explicit validation with clear error message:

```diff
function claimDividends() external {
    address user = msg.sender;
+   require(block.timestamp >= startTime, "Dividend vesting has not started yet");
    
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
    uint256 claimableAmount = (totalAmount * claimableSeconds) / vestingPeriod;
    stablecoin.safeTransfer(user, claimableAmount);
    emit dividendsClaimed(user, claimableAmount);
}
```
  