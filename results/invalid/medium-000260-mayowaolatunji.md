# [000260] Changing vestingPeriodDivisor May Break updateBid() Logic
  
  ### Summary

The contract allows the owner to freely modify the global [vestingPeriodDivisor ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L397) through setVestingPeriodDivisor(). This value is later used inside updateBid() to enforce that any new vesting period chosen by a bidder must be a clean multiple of the divisor. Because the owner can change this divisor at any time—including during an active bidding period—users who previously selected valid vesting periods may suddenly be unable to update their bids. This creates a state where users are unintentionally locked out of bid adjustments, disrupting fair participation and potentially altering allocation outcomes.

```
if (newVestingPeriod > 1) {
    require(
        newVestingPeriod % vestingPeriodDivisor == 0,
        Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
    );
}

```

### Root Cause

The core issue is that vestingPeriodDivisor is a mutable global parameter that affects live user bidding logic, but the contract does not restrict when this parameter may be changed. When the owner updates the divisor during an active bidding window, existing users’ vesting periods may no longer satisfy the modulo condition required by updateBid().

```
function setVestingPeriodDivisor(uint256 newVestingPeriodDivisor) external onlyOwner returns (bool) {
        require(newVestingPeriodDivisor > 0, Zero_Value());
        uint256 oldVestingPeriodDivisor = vestingPeriodDivisor;
        require(
            newVestingPeriodDivisor != vestingPeriodDivisor,
            Same_Value()
        ); //@audit What happens when owner decrement/increment this value during vesting period/claiming tokens. In this case usermay not be able to updateBid
        vestingPeriodDivisor = newVestingPeriodDivisor;
        emit vestingPeriodDivisorUpdated(oldVestingPeriodDivisor, newVestingPeriodDivisor);
        return true;
    }
```

### Internal Pre-conditions

These must hold for the vulnerability to manifest:

- The bidding period for the project is open (startTime ≤ now ≤ endTime and !closed).

- At least one user has an active bid with a vesting period compatible with the old vestingPeriodDivisor.

- The owner sets a new divisor that does not divide the user’s intended new vesting period.

- The user attempts to call updateBid() while the new divisor is in effect.

### External Pre-conditions

_No response_

### Attack Path

1. A user submits a valid bid (e.g., vestingPeriod = 20 when vestingPeriodDivisor = 10).
 
2. The user later attempts to increase their bid amount or extend their vesting period via updateBid().

3. Before the user submits the update, the owner updates the divisor—for example, from 10 to 12.

The user calls:

```
updateBid(projectId, newAmount, newVestingPeriod);
```

4. The contract reaches the divisor check:

```
newVestingPeriod % vestingPeriodDivisor == 0
```

5. Because 20 % 12 ≠ 0, the transaction reverts.

6. The user is now unable to update their bid for the remainder of the bidding period—even though their original vesting schedule was valid.

This results in a soft DoS, where users lose the ability to modify legitimate bids due solely to an owner-controlled parameter change.

### Impact

Users who previously submitted valid bids may be unable to update their bids if the owner modifies the vestingPeriodDivisor during the bidding phase. This creates a soft denial‑of‑service affecting all bidders whose vesting periods no longer satisfy the updated modulo requirement.

### PoC

_No response_

### Mitigation

The simplest and safest mitigation is to restrict updates to vestingPeriodDivisor so they cannot occur during active bidding periods. This prevents the divisor from invalidating user actions mid-process.

```solidity
function setVestingPeriodDivisor(uint256 newVestingPeriodDivisor)
    external
    onlyOwner
{
    require(!biddingActive(), "Cannot modify divisor during active bidding");
    // ...rest of logic
}

function biddingActive() internal view returns (bool) {
    for (uint256 i = 0; i < biddingProjectCount; i++) {
        BiddingProject storage p = biddingProjects[i];
        if (block.timestamp >= p.startTime && block.timestamp <= p.endTime && !p.closed) {
            return true;
        }
    }
    return false;
}
```
  