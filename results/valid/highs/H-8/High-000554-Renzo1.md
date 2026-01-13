# [000554] Infinite Loop in `getUnclaimedAmounts` Due to Missing Loop Counter Increment Before Continue Statements Causes Gas Exhaustion and Function Reversion
  
  
## Summary

Missing loop counter increment before `continue` statements in `A26ZDividendDistributor::getUnclaimedAmounts` causes an infinite loop that exhausts gas and reverts. This breaks the function for NFTs with claimed flows or zero claimed seconds, preventing users from querying unclaimed amounts and blocking owner functions that depend on this calculation.

```js
    function getUnclaimedAmounts(...) public returns (uint256 amount) {
        // ...
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
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

## Root Cause

In [`A26ZDividendDistributor.sol:147-159`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L158), the loop increments `i` at lines 156-158, but two `continue` statements at lines 148 and 151 skip that increment. When `claimedFlows[i]` is true (line 148) or `claimedSeconds[i] == 0` (line 151), execution jumps to the next iteration without incrementing `i`, causing an infinite loop.

## Internal Pre-conditions

1. An NFT must exist with at least one flow where `claimedFlows[i] == true`, OR
2. An NFT must exist with at least one flow where `claimedSeconds[i] == 0`

## External Pre-conditions

None required. The bug triggers based on existing NFT state.

## Attack Path

1. An NFT is created or updated to a state with vesting allocations where some flows have `claimedFlows[i] == true` or `claimedSeconds[i] == 0`
2. A user or owner calls `getUnclaimedAmounts(nftId)` for that NFT
3. The loop reaches an index `i` where `claimedFlows[i] == true` or `claimedSeconds[i] == 0`
4. The `continue` statement executes, skipping the increment at lines 156-158
5. The loop condition `i < len` remains true, and `i` never increments
6. The loop runs indefinitely until gas is exhausted, causing the transaction to revert

## Impact

Users cannot query unclaimed amounts for affected NFTs. Owner functions that call `getUnclaimedAmounts` (e.g., `getTotalUnclaimedAmounts()` at line 131, which is used by `_setAmounts()` at line 209) will revert, blocking dividend setup and distribution. This can prevent dividend distribution and lock protocol functionality.

## Proof of Concept
We use a minimal version of `getUnclaimedAmount` that just focuses on the bug. Add the following code in any test files to execute the test
```js
    // forge test --match-test test_TestInfiniteLoop -vv
    function test_TestInfiniteLoop() public {
        
        // This should revert because _minimalGetUnclaimedAmounts will encounter an OOG error due to infinite loop
        vm.expectRevert();
        this._minimalGetUnclaimedAmounts(3);
    }


    function _minimalGetUnclaimedAmounts(uint256 length) public pure returns (bool success) {
        for (uint256 i; i < length;) {
            if(i == 1){
                continue;
            }

            unchecked {
                ++i;
            }
        }
        success = true;
    }
```

If you comment out the `vm.expectRevert()`, it fails with an EvmError revert:

```bash
[FAIL: EvmError: Revert] test_TestInfiniteLoop() (gas: 1056943918)
```

## Mitigation

Increment the loop counter before each `continue` statement. Move the increment into the conditional blocks that use `continue`:

```diff
for (uint i; i < len;) {
    if (claimedFlows[i]) {
+        unchecked {
+            ++i;
+        }
        continue;
    }
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
+        unchecked {
+            ++i;
+        }
        continue;
    }
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked {
        ++i;
    }
}
```

Alternatively, use a traditional for loop with increment in the header, or restructure to avoid `continue` statements.
  