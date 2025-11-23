# [000076] Changing vestingPeriodDivisor Causes DoS for Existing Bidders
  
  ### Summary

Changing the `vestingPeriodDivisor` after users have placed bids will cause a DoS for those users when they attempt to update their bids, as the `updateBid()` function validates the existing vestingPeriod against the new divisor, causing valid bids to fail the modulo check.

### Root Cause

In [AlignerzVesting.sol:755-758](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L755-L759), the `updateBid()` function validates newVestingPeriod against the current `vestingPeriodDivisor` using a modulo check. However, when the owner changes the divisor via `setVestingPeriodDivisor()`, existing bids that were valid under the old divisor may no longer satisfy the new divisor's modulo requirement.

### Internal Pre-conditions

1. Owner needs to call `setVestingPeriodDivisor()` to change the divisor from one value (e.g., `2_592_000`) to another (e.g., `5_184_000`) 
2. Users need to have existing bids with vestingPeriod values that were valid under the old divisor but are not multiples of the new divisor 

### External Pre-conditions

None

### Attack Path

1. User calls placeBid() with vestingPeriod = `7_776_000` (90 days) when vestingPeriodDivisor = `2_592_000` (30 days) 
2. Bid is accepted because `7_776_000 % 2_592_000 == 0` 
3. Owner calls `setVestingPeriodDivisor(5_184_000)` to change divisor to 60 days 
4. User attempts to call `updateBid()` with newVestingPeriod = `7_776_000` (keeping same vesting period) and `newAmount > oldAmount` 
5. Transaction reverts with ` Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()` because `7_776_000 % 5_184_000 != 0` 
6. User cannot update their bid amount without changing their vesting period to comply with the new divisor

### Impact

Users with existing bids cannot update their bid amounts (even to increase them) without being forced to change their vesting periods to comply with the new divisor. This creates a DoS condition where users must either accept a different vesting period than originally intended or forfeit the ability to increase their bid amount. The protocol loses flexibility and users lose the ability to adjust their bids during the bidding period.

### PoC

N/A

### Mitigation

Store the `vestingPeriodDivisor` value at the time of bid placement in the `Bid` stuct, and validate against it instead of using the new `vestingPeriodDivisor`.

```diff
struct Bid {  
    uint256 amount;  
    uint256 vestingPeriod;  
+    uint256 divisorAtPlacement;   
}  
  
function updateBid(...) external {  
    // Validate against the divisor that was active when bid was placed  
    if (newVestingPeriod > 1) {  
        require(  
+            newVestingPeriod % bid.divisorAtPlacement == 0,  
            Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()  
        );  
    }  
}
```

  