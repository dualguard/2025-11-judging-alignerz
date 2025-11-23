# [000352] [M-1] Missing loop increment in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS` will make the loop run infinitely and revert
  
  ### Summary

A missing loop increment in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS` will cause an infinite loop, this will results in the function reverting causing a disruption to the protocol.

### Root Cause

In [`FeesManager.sol::calculateFeeAndNewAmountForOneTVS#173`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169), the `for` loop does not increment the counter `i`, causing an infinite loop.

### Internal Pre-conditions

1. A call is made to `FeesManager.sol::calculateFeeAndNewAmountForOneTVS`.
2. The `length` parameter is greater than `0`.

### External Pre-conditions

_No response_

### Attack Path

1. An attacker calls `calculateFeeAndNewAmountForOneTVS` function.
2. This function executes the loop: `for (uint256 i; i < length; )`.
3. Because `i` never increments, the loop runs infinitely.
4. The transaction always reverts because the block gas limit is exceeded.
5. Any feature that depends on this function becomes unusable, causing a major disruption to the protocol.

### Impact

This causes a complete halt in `FeesManager.sol::calculateFeeAndNewAmountForOneTVS`, and any dependent logic causing complete disruptions to the protocol.
For example both `AlignerzVesting.sol::mergeTVS#1002` and `AlignerzVesting.sol::splitTVS#1055` calls this as an internal logic, making calculation of fees imposible.

### PoC

```solidity
// Place the following into `AlignerzVestingProtocolTest.t.sol`
// FeesManager is an abstarct contract
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

function test_InfiniteLoop() public {
    uint256[] memory amounts = new uint256[](2);
    amounts[0] = 50;
    amounts[1] = 50;

    // This call always reverts
    calculateFeeAndNewAmountForOneTVS(50, amounts, 2);
}

contract FeesManagerImpl is FeesManager {
}
```

### Mitigation

```diff
    function calculateFeeAndNewAmountForOneTVS(...) public pure returns (...) {
-       for (uint256 i; i < length; ) {
+       for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  