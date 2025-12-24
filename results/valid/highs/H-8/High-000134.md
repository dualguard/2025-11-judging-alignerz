# [000134] Infinite loop in `getUnclaimedAmounts` causes gas denial-of-service during dividend setup
  
  ### Summary

The `getUnclaimedAmounts()` function in the A26ZDividendDistributor contract contains an infinite loop vulnerability caused by early `continue` statements that skip the loop increment. This causes any call with certain normal allocation states to exhaust gas and revert, blocking dividend distribution operations. The severity is medium because the condition occurs during normal operations and reliably prevents dividend setup.

### Root Cause

The `getUnclaimedAmounts()` function iterates over allocation flows to calculate unclaimed token amounts. The loop is declared with a manual increment in the body:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L152

The problem occurs when either `claimedFlows[i]` is true or `claimedSeconds[i]` equals zero. In both cases, the `continue` statement jumps to the next iteration without executing the `unchecked { ++i; }` increment. Since the `for` loop header contains no post-expression, the index `i` is never incremented on these paths, causing the loop to repeat infinitely on the same iteration.

These states are normal during dividend setup: fresh allocations have `claimedSeconds[i] == 0`, and fully vested flows have `claimedFlows[i] == true`. Consequently, any NFT with these characteristics triggers the infinite loop.

### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

1. Protocol owner calls `setUpTheDividends()` or `setAmounts()` to initialize dividend distribution
2. Function invokes `getTotalUnclaimedAmounts()` to calculate total unclaimed amounts
3. `getTotalUnclaimedAmounts()` iterates through minted NFTs and calls `getUnclaimedAmounts(nftId)` for each
4. When an NFT with a flow marked as claimed or with zero claimed seconds is encountered, `getUnclaimedAmounts()` enters an infinite loop
5. Transaction runs out of gas and reverts, blocking dividend setup

### Impact

A DOS occurs because of the infinite loops, breaking a core functionality of the protocol.

### PoC

_No response_

### Mitigation

Consider restructuring the loop to ensure the increment always executes. The most straightforward approach is to move the increment to the top of the loop or use a different loop structure:
  