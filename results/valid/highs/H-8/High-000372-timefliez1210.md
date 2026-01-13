# [000372] H06 Infinite loop in getUnclaimedAmounts in A26ZDividendDistributor causes DoS
  
  ### Summary

Within `A26ZDividendDistributor::getUnclaimedAmounts` the function iterates through vesting allocations to calculate unclaimed amounts. However, conditional `continue` statements bypass the loop increment, causing an infinite loop if specific conditions are met.

### Root Cause

Looking at `A26ZDividendDistributor::getUnclaimedAmounts`:

```solidity
    function getUnclaimedAmounts(
        uint256 nftId
    ) public returns (uint256 amount) {
        /// *** SNIP *** ///
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len; ) {
1. @>       if (claimedFlows[i]) continue;
2. @>       if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) /
                vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
3. @>           ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

We can see at markers 1 and 2 that if `claimedFlows[i]` is true or `claimedSeconds[i]` is 0, the `continue` keyword is used. This jumps to the next iteration of the loop *without* executing the increment at marker 3. Since `i` is never incremented, the loop condition `i < len` remains true forever (assuming `len > 0`), resulting in an infinite loop and an Out of Gas revert.

### Internal Pre-conditions

The `nftId` must have an allocation with at least one flow where `claimedFlows[i]` is true OR `claimedSeconds[i]` is 0.


### External Pre-conditions

None.

### Attack Path

It's a bug, not an attack.

### Impact

This causes a Denial of Service (DoS) for the dividend distribution mechanism. The owner cannot set dividends (`_setDividends`), and the protocol cannot calculate total unclaimed amounts. Since `claimedSeconds` starts at 0 for all allocations, this function is broken from the start for any new allocation. Likelihood is High (certainty), and Impact is High (broken core functionality).

### PoC

Trivial. Any allocation with `claimedSeconds[i] == 0` or `claimedFlows[i] == true` triggers the infinite loop.


### Mitigation

Ensure the loop counter is incremented before `continue` or use a standard for-loop structure.

```diff
    function getUnclaimedAmounts(
        uint256 nftId
    ) public returns (uint256 amount) {
        // ...
        uint256 len = vesting.allocationOf(nftId).amounts.length;
-       for (uint i; i < len; ) {
+       for (uint i; i < len; ++i) {
            // @audit will the ++i ever be reached if claimedSeconds == 0 or claimedFlows[i] == true?
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) /
                vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
-           unchecked {
-               ++i;
-           }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

  