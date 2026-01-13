# [000915] Infinite Loop in getUnclaimedAmounts() Due to Missing Loop Increment
  
  ### Summary

`getUnclaimedAmounts(uint256 nftId)` contains a broken loop that results in an **infinite loop** whenever unclaimed vesting flows exist. Because `i` is never incremented in several execution paths, the function never terminates and always reverts, breaking all dividend-related logic that depends on it.

### Root Cause

Inside [`getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L152) there is a loop:

```solidity
for (uint256 i; i < len;) {
```

the function contains two `continue` statements:

```solidity
if (claimedFlows[i]) continue;

if (claimedSeconds[i] == 0) {
    amount += amounts[i];
    continue;
}
```

Neither branch increments `i`, so if a continue is hit, the loop repeatedly processes the same index. This results in an infinite loop → out-of-gas → revert.

### Internal Pre-conditions

There contains a NFT with `claimedFlows` set as true, or if `claimedSeconds == 0`. 

> Note: `claimedSeconds == 0` will always be true for first time claimers

### External Pre-conditions

N/A

### Attack Path

1. Owner calls `setUpTheDividends`.
2. `_setAmounts` calls `getTotalUnclaimedAmounts`.
3. For each NFT, `getUnclaimedAmounts` is invoked.
4. Since `claimedSeconds[0] == 0`, the code hits:
    ```solidity
    amount += amounts[i];
    continue; // i never increases
    ```
5. Loop restarts with the same `i = 0`.
6. Execution never progresses → infinite loop → out-of-gas → revert.

### Impact

`getUnclaimedAmounts` always reverts due to OOG (if other bugs in this function is fixed), therefore, dividends cannot be set or claimed by TVS holders. Therefore, users may receive **zero dividends** even if they hold valid allocations.

### PoC

I created a standalone PoC that isolates the `getTotalUnclaimedAmounts` logic, because the full system has multiple interacting bugs that prevent reliably reaching this code path in its deployed form.

Add the following test suite into `test/getUnclaimedAmounts.t.sol` and run the test

```solidity
forge clean
forge test --mt test_POC_ClaimedFlow_Stuck -vvvv
```

You will see that it will revert with `EvmError: OutOfGas`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

//-------------------------------------------------------------
// Mock interfaces / structs to isolate getUnclaimedAmounts
//-------------------------------------------------------------
contract MockVesting {
    struct Allocation {
        address token;
        uint256[] amounts;
        uint256[] claimedSeconds;
        uint256[] vestingPeriods;
        bool[] claimedFlows;
    }

    mapping(uint256 => Allocation) public allocations;

    function setAllocation(
        uint256 nftId,
        address token,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory periods,
        bool[] memory flows
    ) public {
        allocations[nftId] = Allocation(token, amounts, claimedSeconds, periods, flows);
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return allocations[nftId];
    }
}

//-------------------------------------------------------------
// Dividend Distributor Logic (Isolated PoC Version)
//-------------------------------------------------------------
contract UnclaimedAmountsPOC {
    MockVesting public vesting;
    address public token;

    mapping(uint256 => uint256) public unclaimedAmountsIn;

    constructor(address _vesting, address _token) {
        vesting = MockVesting(_vesting);
        token = _token;
    }

    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint256 i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
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
}

//-------------------------------------------------------------
// PoC Test
//-------------------------------------------------------------
contract GetUnclaimedAmountsPOCTest is Test {
    MockVesting vesting;
    UnclaimedAmountsPOC poc;
    address TOKEN = address(0xBEEF);

    function setUp() public {
        vesting = new MockVesting();
        poc = new UnclaimedAmountsPOC(address(vesting), TOKEN);
    }

    function test_POC_ClaimedFlow_Stuck() public {
        // Create flow: claimedFlow = false → triggers infinite loop / stuck loop
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1000;

        uint256[] memory secondsClaimed = new uint256[](1);
        secondsClaimed[0] = 0; // triggers the branch that doesn't increment i

        uint256[] memory periods = new uint256[](1);
        periods[0] = 100;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false; // <-- KEY TRIGGER FOR VULN

        vesting.setAllocation(
            1,
            address(0x123), // different from distributor token → skip the early return (different bug)
            amounts,
            secondsClaimed,
            periods,
            claimedFlows
        );

        vm.expectRevert(); // function never increments i → infinite loop → OOG revert
        poc.getUnclaimedAmounts(1);
    }
}
```

### Mitigation

Since there are multiple continues, just cache it. Ensure to always increment `i` every iteration.

```solidity
for (uint256 i; i < alloc.amounts.length; ++i)
```
  