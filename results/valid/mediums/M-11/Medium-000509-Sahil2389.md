# [000509] Incorrect Stablecoin Amount Calculation Includes Unclaimed Dividends, Leading to Double-Counting and Insolvency
  
  ### Summary

The `_setAmounts()` function sets `stablecoinAmountToDistribute` to the entire contract balance without excluding already-reserved unclaimed dividends. This causes the same funds to be counted multiple times across distribution rounds, inflating user dividend balances beyond available reserves and making claims impossible.


### Root Cause

In `A26ZDividendDistributor.sol` contract
In `_setAmounts()`:
```solidity
stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
```
This counts all stablecoins in the contract, including amounts already allocated to users from previous distributions. The correct logic should be:
```solidity
stablecoinAmountToDistribute = stablecoin.balanceOf(address(this)) - totalReservedForDistribution;
```
The contract lacks a `totalReservedForDistribution` variable to track funds already allocated but unclaimed, causing previously allocated dividends to be redistributed again.

Code Snippet- 
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211


### Internal Pre-conditions

1. Contract must have unclaimed dividends from a previous distribution.
2. Owner calls `setUpTheDividends()` multiple times(min 2).


### External Pre-conditions

1. Owner deposits additional stablecoins for a new distribution round.
2. Users haven't fully claimed dividends from previous rounds.
3. Owner calls `setUpTheDividends()` again.
4. when user going to withdraw then vesting period already passed.


### Attack Path

1. Round 1: Owner deposits 100 stablecoins, calls `setUpTheDividends()`
   - Contract balance: 100
   - `stablecoinAmountToDistribute = 100`
   - `dividendsOf[user].amount = 100`
   - Reserved for distribution in contract 100
   - `totalUnclaimedAmounts = 100` (assuming only one user there for easy understanding purpose)

2. User doesn't claim: Contract balance remains 100 (all reserved)

3. Round 2: Owner deposits 100 more stablecoins, calls `setUpTheDividends()`
   - Contract balance: 200 (100 unclaimed + 100 new)
   - `stablecoinAmountToDistribute = 200`  (should be 100)
   - `totalUnclaimedAmounts = 100`
   - `dividendsOf[user].amount += (100 * 200 / 100) = 100 + 200 = 300`
   -  Reserved for distribution in contract 200 but user is eligible to withdraw 300
  
4.  Result: on this situation whenever user going to claim Dividends function will revert because contract balance is less than User allocated amount (assuming when user going to withdraw  then vesting period already passed).

### Impact

Complete breakdown of dividend distribution mechanism. Last Users lose access to their rightful dividends(because of insufficient balance) when multiple user is there. Protocol becomes mathematically insolvent (liabilities > assets).


### PoC

N/A

### Mitigation

N/A
  