# [001012] Unclaimed dividends from previous rounds are incorrectly included in the next distribution round; Dividends accounting is broken
  
  ### Summary

The `A26ZDividendDistributor::_setAmounts()` function sets `stablecoinAmountToDistribute` to the entire stablecoin balance of the contract without excluding unclaimed dividends from the previous quarter.  The unclaimed amounts get redistributed in the next round, effectively allowing other users to claim rewards that were already allocated to someone else. 

### Root Cause

[_setAmounts()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L206-L211) inclused unclaimed dividends from previous rounds in the distribution for the next `vestingPeriod`/ quarter.

```solidity
    /// @notice Internal logic that allows the owner to set crucial amounts for dividends calculations
    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this)); // @audit includes unclaimed dividends
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }
```
This means that if users haven't fully claimed their dividends from a previous round, those unclaimed amounts get [redistributed](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218C92-L218C93) in the next round, effectively allowing other users to claim rewards that were already allocated to someone else. This violates the fairness principle of the dividend distribution system and can lead to users losing their rightful share of rewards.

```solidity
    function _setDividends() internal {
//...
            // @audit unclaimed dividends are redistributed 
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
//...
    }
```

### Internal Pre-conditions

1. Admin set up the dividends  for at least a second quarter. 

### External Pre-conditions

_No response_

### Attack Path

1. Admin sets up the dividends distribution for quarter1
2. At the end of quarter1/ start of quarter2 at least 1 user (Alice)  has unclaimed dividends 
3. `vestingPeriod` had passed and Admin sets up dividends for the quarter2: 
- transfer stablecoin rewards to `A26ZDividendDistributor`
-  calls `_setAmounts()`: the `stablecoinAmountToDistribute` includes the Alice's unclaimed dividends. Her rewards gets redistributed.

### Impact


Users who don't claim their dividends will have their unclaimed amounts redistributed to others in subsequent rounds, resulting in a loss of rewards.

### PoC

_No response_

### Mitigation

When a new quarter starts, set the `stablecoinAmountToDistribute` to the amount of tokens transferred. 
Consider updating `_setAmounts()` and use `transferFrom()` method to pull `rewardsAmount` from  msg.sender. Set the `stablecoinAmountToDistribute` to `rewardsAmount` value.
  