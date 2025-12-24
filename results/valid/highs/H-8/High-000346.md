# [000346] Incorrect early `continue` in `A26ZDividendDistributor.sol::getUnclaimedAmounts#140` causes an infinite loop risk
  
  ### Summary

A missing index increment before `continue` will cause the loop to get stuck at the same index, causing unexpected reverts or gas exhaustion.

### Root Cause

In the loop in [`A26ZDividendDistributor.sol::getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147),  when `claimedFlows[i] == true`, the `continue` statement is executed without incrementing `i`, the loop will never terminate (an infinite loop).

### Internal Pre-conditions

1. If `claimedFlows[i] == true`.
2. Execution reaches the loop.
3. `i` is never incremented.

### External Pre-conditions

_No response_

### Attack Path

1. Any caller or other function invokes `getUnclaimedAmounts`.
2. For index `i` where `claimedFlows[i] == true`, execution loops infinitely.

### Impact

Users experience a complete function DoS, blocking vesting accounting. And this is a complete disruption to the protocol.

### PoC

_No response_

### Mitigation

To fix this, `i` should be incremented on every iteration of the loop.

```solidity
       // We can do this by either moving i to the header of the loop
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // ...
        for (uint i; i < len; i++) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
           // ...
        }
        // ...
    }

// Or we can also do it by incrementing `i` inside each (if-conditional) body, before the continue statement.
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // ...
        for (uint i; i < len;) {
            if (claimedFlows[i]) {
                unchecked {
                    ++i;
                }
                continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                unchecked {
                    ++i;
                }
                continue;
            }
            // ...
            unchecked {
                ++i;
            }
        }
        // ...
    }
```
  