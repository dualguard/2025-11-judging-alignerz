# [000355] Rounding-Down in Claimable Calculations Causes Systematic Underpayment of Users Across Dividends and TVS Allocations claiming
  
  ### Summary

Multiple vesting-related calculations in the protocol compute a user’s claimable amount using the formula:

```
claimableAmount = amount * claimableSeconds / vestingPeriod
```

Because Solidity integer division **rounds down**, any fractional results are truncated. This results in **systematic underpayment** to users whenever:

* `claimableSeconds * amount` is not perfectly divisible by `vestingPeriod`, or
* Claims are made in small increments, or
* Users have small dividends / vesting allocations.

The result is that users **never receive the full vested amount**, as the fractional dust is lost on every calculation. This affects both the general TVS allocation claiming flow and the dividend-claiming flow.



### Root Cause

### **1. TVS Allocation Claiming**

```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;      // rounding down. claimableseconds as a fraction will cause issues e.g 1.9 1.8 is 1
require(claimableAmount > 0, No_Claimable_Tokens());   // this reduces the impact
```

For allocation 0 claimable is prevented 

### **2. Dividends Claiming**

```solidity
claimableAmount = totalAmount * claimableSeconds / vestingPeriod; // rounding down // can round down to 0 depending on divisor value and total amount 

// there is no check to prevent 0 claimable like TVS allocation
```

While this is not so for dividend claiming.  There is no check to prevent 0 claimable like TVS allocation

```solidity

   secondsPassed = block.timestamp - startTime;

@audit>>   dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
```

Claimable seconds will be updated regardless, leading to 0 dividends collected in some cases.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L200-L201

E.g user has a dividend of 6 usdt - vesting period is about 3 month. 

claimable in 1 seconds = 6 *10**6 * 1 /  7257600 = 0.8267195767 = 0 in solidity , this is not prevented in dividend claiming hence loss of dividends for the user.


### Internal Pre-conditions

1. allocation/dividends set


### External Pre-conditions

_No response_

### Attack Path

1. Claiming of dividends 
2. Precision loss to 0

### Impact

Each time a user claims, the fractional amount due to rounding down is **lost**, accumulating over time. For dividends, users can get 0 for claiming each second under some conditions. 

### PoC

_No response_

### Mitigation

Redesign the claiming formula to ensure that all dividends and allocations are claimed without any precision error. 
  