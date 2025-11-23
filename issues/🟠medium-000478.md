# [000478] FeesManager::calculateFeeAndNewAmountForOneTVS always reverts due to an out-of-bounds write
  
  ### Summary

FeesManager.calculateFeeAndNewAmountForOneTVS contains two critical bugs:
	1.	newAmounts is never allocated, so any write to newAmounts[i] causes an out-of-bounds memory write and immediate revert.
	2.	The for loop omits i++, making the loop impossible to exit (infinite loop), which also guarantees revert.

As a result, all calls to this function will revert, and therefore any features relying on it—specifically mergeTVS() and splitTVS()—will always revert as well.

### Root Cause

1. newAmounts Is Never Allocated

The function returns newAmounts but never initializes it. The code then writes to newAmounts[i], which always triggers an out-of-bounds write → revert.

2. The for Loop Is Missing i++

The loop never increments i, so it will never terminate and will always revert due to out-of-gas.

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Internal Pre-conditions

always happens

### External Pre-conditions

_No response_

### Attack Path

1. user calls `mergeTVS` or `splitTVS`
2. revert

### Impact

Users cannot merge or split their TVs

### PoC

FeesManagerBugsPoC.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {FeesManager} from "../src/contracts/vesting/feesManager/FeesManager.sol";

// Concrete implementation of abstract FeesManager for testing
contract TestableFeesManager is FeesManager {
    function initialize() public initializer {
        __Ownable_init(msg.sender);
        __FeesManager_init();
    }
}

contract FeesManagerBugPoC is Test {
    TestableFeesManager public feesManager;
    
    function setUp() public {
        feesManager = new TestableFeesManager();
        feesManager.initialize();
    }
    
    function test_UninitializedArrayBug() public {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1000;
        amounts[1] = 2000;
        
        // This will revert because newAmounts is never initialized
        vm.expectRevert();
        feesManager.calculateFeeAndNewAmountForOneTVS(100, amounts, 2);
    }
}
```

```
forge test --match-test test_UninitializedArrayBug -vv
```

### Mitigation


```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+    newAmounts = new uint256[](length); <== add this line
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
+          i++; <==
        }
```
  