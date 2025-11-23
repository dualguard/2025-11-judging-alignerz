# [000254] Infinite Loop DoS in `A26ZDividendDistributor::getUnclaimedAmounts` Due to Missing Index Increment
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L151

The `A26ZDividendDistributor::getUnclaimedAmounts` function enters an infinite loop when certain conditions are met because the loop index `i` is never incremented after a continue statement. This allows any user to permanently freeze calls to `A26ZDividendDistributor::getUnclaimedAmounts`, blocking dependent flows such as claiming, updating vesting info, or any feature that relies on this function.

### Root Cause

Inside the loop:
```solidity
for (uint i; i < len;) {
    if (claimedFlows[i]) continue;
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue;
    }
    ...
    ++i;
}
```
The `continue` statements skip the increment operation `++i`.
So if any index satisfies either:
```sol
claimedFlows[i] == true
```
or
```sol
claimedSeconds[i] == 0
```
the loop repeatedly executes the same iteration, causing an infinite loop. The `continue` statement skips the rest of the current loop iteration and jumps straight to the next iteration’s condition check and since `i` was not incremented, it loops indefinitely.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- A user mints or uses an NFT with a vesting entry that sets:
```sol
claimedFlows[i] = true
or
claimedSeconds[i] = 0
```
- They then trigger:
```sol
getUnclaimedAmounts(nftId)
```
- The loop hits a continue before reaching:
```sol
++i;
```
- The function never increments `i`, so it loops indefinitely until running out of gas.

All future calls to `getUnclaimedAmounts(nftId)` revert due to out of gas, creating a permanent DoS on that NFT.

### Impact

- Permanent denial of service for any nftId with those entries.
- Any function relying on `getUnclaimedAmounts` becomes unusable.
- Users cannot claim or update vesting data for affected NFTs.

### PoC

Create a test file in `protocol/test` folder and paste the below, run test with `forge test --match-test test_InfiniteLoop_DoS_in_getUnclaimedAmounts -vv`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

contract MinimalVesting {
    struct Allocation {
        uint256[] amounts;
        uint256[] claimedSeconds;
        uint256[] vestingPeriods;
        bool[] claimedFlows;
        address token;
    }

    mapping(uint256 => Allocation) public allocationOf;
    mapping(uint256 => uint256) public unclaimedAmountsIn;

    address public token = address(0xBEEF);

    function pushAmount(uint256 id, uint256 x) external {
        allocationOf[id].amounts.push(x);
    }

    function pushVesting(uint256 id, uint256 x) external {
        allocationOf[id].vestingPeriods.push(x);
    }

    function pushClaimedSecond(uint256 id, uint256 x) external {
        allocationOf[id].claimedSeconds.push(x);
    }

    function pushClaimedFlow(uint256 id, bool x) external {
        allocationOf[id].claimedFlows.push(x);
    }

    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (token == allocationOf[nftId].token) return 0;

        uint256[] memory amounts = allocationOf[nftId].amounts;
        uint256[] memory claimedSeconds = allocationOf[nftId].claimedSeconds;
        uint256[] memory vestingPeriods = allocationOf[nftId].vestingPeriods;
        bool[] memory claimedFlows = allocationOf[nftId].claimedFlows;

        uint256 len = amounts.length;

        for (uint256 i; i < len;) {
            // continue without increment, infinite loop if condition hits
            if (claimedFlows[i]) continue;

            // second infinite loop path
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }

            uint256 claimedAmount =
                (claimedSeconds[i] * amounts[i]) / vestingPeriods[i];
            amount += amounts[i] - claimedAmount;

            unchecked {
                ++i;
            }
        }

        unclaimedAmountsIn[nftId] = amount;
    }
}

contract MinimalVestingTest is Test {
    MinimalVesting vest;

    function setUp() public {
        vest = new MinimalVesting();
    }

    function test_InfiniteLoop_DoS_in_getUnclaimedAmounts() public {
        uint256 nftId = 1;

        // Prepare a single vesting entry
        vest.pushAmount(nftId, 100);
        vest.pushVesting(nftId, 1000);

        // Either condition creates a permanent infinite loop
        vest.pushClaimedFlow(nftId, true);
        vest.pushClaimedSecond(nftId, 0);

        // Since `i` never increments, this call never terminates, OOG revert
        vm.expectRevert();
        vest.getUnclaimedAmounts(nftId);
    }
}
```

### Mitigation

- Increment the loop index before each continue:
```solidity
for (uint256 i; i < len;) {
    if (claimedFlows[i]) {
        unchecked { ++i; }
        continue;
    }

    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        unchecked { ++i; }
        continue;
    }

    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    amount += amounts[i] - claimedAmount;

    unchecked { ++i; }
}
```
  