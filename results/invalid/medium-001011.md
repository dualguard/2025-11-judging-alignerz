# [001011] Broken dividend accounting due to `startTime` update issues
  
  ### Summary

When transitioning between vesting periods (3 months/ quarters), the contract fails to properly handle the `startTime` variable, leading to broken dividend calculations regardless of whether the admin updates it or not. This affects the `claimDividends()` function, which uses `startTime` to calculate vested amounts for users.

### Root Cause


 The [claimDividends()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187-L204) function calculates secondsPassed as:

```solidity
    function claimDividends() external {
///...
        if (block.timestamp >= vestingPeriod + startTime) {
@>                    secondsPassed = vestingPeriod;
                    dividendsOf[user].amount = 0;
                    dividendsOf[user].claimedSeconds = 0; // full vested
                } else {
@>                    secondsPassed = block.timestamp - startTime;
                    dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
                }
                uint256 claimableSeconds = secondsPassed - claimedSeconds;
                // 3 months vesting = 7_776_000 seconds
````

Even if the `startTime` is updated or not, the dividend accounding is broken.
1. Case 1: `startTime` is not updated
- in this case when users calls `claimDividends` the `block.timestamp` will be bigger than `vestingPeriod + startTime`. `secondPassed` is set to `vestingPeriod`, resulting in entire reward to be vested is claimable immediately. 
```solidity
    function claimDividends() external {
//...
        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;                //@audit user can claim full amount if `startTime` is not updated
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
//..
        }
        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
        emit dividendsClaimed(user, claimableAmount);
    }
```
2. Case 2: admin updates `startTime`
- if `startTime` is updated to reflect the new vesting period start time, users with unclaimed dividends from previous quarters will not be able to claim their full dividends. Since `startTime` was updated to reflect the new vesting period, users with unclaimed vested dividends must wait again `vestingPeriod` in order to claim old dividends. 
```solidity
    function claimDividends() external {

        uint256 claimedSeconds = dividendsOf[user].claimedSeconds; // @audit rely on previous claimed seconds
//...
        if (block.timestamp >= vestingPeriod + startTime) { // @audit the 'if' will be false due to updated startTime
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
            secondsPassed = block.timestamp - startTime; // @audit old unclaimed but vested dividends are not fully claimable
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }
    }
```

Users will be able to claim all dividends (old + new) after the `vestingPeriod` for the second distribution has passed.
If user partially claimed in previously quarter (but not fully => `claimedSeconds`  is not reset): 
- the `claimDividends()` tx reverts due to underflow in `claimableSeconds = secondsPassed - claimedSeconds` AND
- after `secondsPassed` will be bigger than old `claimedSeconds`, user  receive dividends corresponing to ` secondsPassed - claimedSeconds`, even if the `claimedSeconds` is from previous quarter. 

### Internal Pre-conditions

1. A first `vestingPeriod` has passed and Admin sets up the dividend distribution for the next period. 

### External Pre-conditions

None

### Attack Path

The execution flow for the vulnerable paths are explained in Root Cause section.

### Impact

Dividends do not vest independently for each quarter. Once the first period is complete, all later distributions either:
- become instantly claimable (if `startTime` is not updated), or
- are vested using a time reference (`startTime`) that conflicts with already-recorded `claimedSeconds` (if `startTime` is updated) => users don't get their full dividends.

### PoC

_No response_

### Mitigation

The contract needs a redesign: all dividend details (vesting period, start time, claimed seconds, amount) should be tracked individually for each `vestingPeriod` for each user. 
  