# [000768] Missing loop increments in `getUnclaimedAmounts` cause infinite loop 
  
  ### Summary

Not incrementing the loop index `i` before `continue` in the `claimedFlows[i]` and `claimedSeconds[i] == 0` branches of `getUnclaimedAmounts` will cause infinite loops  whenever those conditions are hit, so any call that processes such a flow will DoS `getUnclaimedAmounts`, `getTotalUnclaimedAmounts`, and thus dividend setup.



### Root Cause


```solidity
// File: A26ZDividendDistributor.sol

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; //@audit should be !=

    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;

    for (uint i; i < len;) {
>>      if (claimedFlows[i]) continue; //@audit infinite loop

>>      if (claimedSeconds[i] == 0) { //@audit infinite loop
>>          if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }

        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;

        unchecked {
            ++i;
        }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

In the `claimedFlows[i]` branch, the loop executes `continue` without incrementing `i`. When `claimedFlows[i]` is true, the next iteration starts with the same `i`, hits the same condition again, and the loop repeats indefinitely until out-of-gas.

In the `claimedSeconds[i] == 0` branch, the code adds `amounts[i]` to `amount` and then executes `continue` without incrementing `i`. As long as `claimedSeconds[i]` remains zero, the loop keeps re-entering the same branch on the same index and never progresses, again leading to an infinite loop and gas exhaustion.


### Internal Pre-conditions

1. There exists at least one flow `i` for some `nftId` where:
   - `claimedFlows[i] == true`, or
   - `claimedSeconds[i] == 0` (the normal initial state for a fresh flow).

2. Dividends are computed by calling `getUnclaimedAmounts(nftId)` directly or via `getTotalUnclaimedAmounts()`.


### External Pre-conditions

1. Admin or any external caller triggers dividend logic (`setUpTheDividends`, `setDividends`, or any wrapper that uses `getTotalUnclaimedAmounts` / `getUnclaimedAmounts`).



### Attack / Failure Path

1. A normal user receives a fresh TVS NFT: for every flow, `claimedSeconds[i] == 0` and `claimedFlows[i] == false`.  
2. When admin first calls `getUnclaimedAmounts(nftId)` or `getTotalUnclaimedAmounts()`, the loop in `getUnclaimedAmounts` enters with `i = 0` and `claimedSeconds[0] == 0`.  
3. It executes the `claimedSeconds[i] == 0` branch, adds `amounts[0]` to `amount`, and hits `continue` **without incrementing `i`**.  
4. On the next iteration, `i` is still `0`, `claimedSeconds[0]` is still `0`, and the same branch runs again → infinite loop until out‑of‑gas.  
5. Result: any call that attempts to include that NFT in dividend calculation will **always revert due to OOG**, DoSing dividend setup for that NFT (and, via `getTotalUnclaimedAmounts`, potentially for everyone).



### Impact

Dividend-related functions that rely on `getUnclaimedAmounts` / `getTotalUnclaimedAmounts` (such as `setUpTheDividends` and `setDividends`) revert with out‑of‑gas as soon as they encounter an NFT whose flows hit these buggy branches.  

This can happen for normal, honest users, because fresh flows naturally start with `claimedSeconds == 0`, so no special attacker interaction is required beyond standard usage.  



### Mitigation

Always increment `i` before any `continue`.


  