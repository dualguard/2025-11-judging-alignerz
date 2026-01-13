# [000129] Multiple dividend rounds with shared `claimedSeconds` silently underpay users
  
  ### Summary

The A26ZDividendDistributor contract miscalculates dividends across multiple distribution rounds by combining newly added principal with stale timing information. `dividendsOf[user].amount` is increased on each `_setDividends()` call, but `dividendsOf[user].claimedSeconds` is not reset, so previously claimed time is re-applied to a larger principal. This underpays users relative to the sum of all configured rounds and can leave excess stablecoins in the contract, which the owner can later withdraw. Severity is high because it systematically causes persistent underpayment of user entitlements.


### Root Cause

The dividend principal per user is accumulated over time:

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint256 i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            dividendsOf[owner].amount +=
                (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        }
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```

Users claim based on a linear vesting schedule:

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

`dividendsOf[user].amount` is the current total principal after all rounds, and `dividendsOf[user].claimedSeconds` tracks how much of the vesting period has already been claimed.

When a new round is configured (`setDividends()` / `setUpTheDividends()`), `dividendsOf[user].amount` increases, but `claimedSeconds` stays the same.

On the next `claimDividends()`, the function uses the new larger `totalAmount` but subtracts `claimedSeconds` from `secondsPassed`, effectively re-applying already-consumed time to the enlarged principal. This means the user does not receive the full incremental benefit of the new distribution.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

This is high severity because it directly and systematically underpays users compared to the sum of all intended dividend rounds. The discrepancy compounds over time and leads to loss of yield for users

### PoC

1. Round 1:
   - `dividendsOf[Alice].amount = 100`, `claimedSeconds = 0`.
   - After half the vesting period, Alice calls `claimDividends()`:
     - `secondsPassed - claimedSeconds = vestingPeriod / 2`.
     - Alice receives `100 * 0.5 = 50`.
     - `claimedSeconds` is updated to `vestingPeriod / 2`.
2. Round 2:
   - Owner calls `setDividends()`, adding another 100 units:
     - `dividendsOf[Alice].amount = 200`, `claimedSeconds = vestingPeriod / 2`.
3. Immediately after, Alice calls `claimDividends()` again:
   - `secondsPassed` is still approximately `vestingPeriod / 2`.
   - `claimableSeconds = secondsPassed - claimedSeconds ≈ 0`.
   - Alice receives almost nothing from the second 100 units, even though a new round was configured.
4. Over the full period, Alice’s total receipts are less than 200, leaving some configured dividends undistributed.

### Mitigation

Consider decoupling per-round vesting from cumulative timing or resetting timing state when a new principal is added. 
  