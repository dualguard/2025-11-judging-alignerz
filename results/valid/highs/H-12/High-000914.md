# [000914] Incorrect TVS Token Check Causes getUnclaimedAmounts to Always Return Zero in A26ZDividendDistributor
  
  ### Summary

The `getUnclaimedAmounts` function in `A26ZDividendDistributor` incorrectly returns zero when the NFT’s allocation token matches the distributor’s TVS token. This occurs due to a flawed early return check:

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

Both `token` in the distributor and `vesting.allocationOf(nftId).token` in `AlignerzVesting` **are meant to be the same TVS token**, so this check **always triggers** for legitimate allocations, causing all unclaimed amounts to return zero.

### Root Cause

The root cause is the [**incorrect conditional check**](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141):

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

- `token` in `A26ZDividendDistributor` is the TVS token (set via `setToken`).
- `vesting.allocationOf(nftId).token` is the NFT’s allocated TVS token.
- Since they are meant to be **the same token**, the current check always returns 0, preventing the correct calculation of unclaimed amounts.

### Internal Pre-conditions

1. `token` in the distributor is set correctly via `setToken` or in the `constructor`.
2. NFT allocations exist in `vesting.allocationOf(nftId)`.

### External Pre-conditions

N/A

### Attack Path

1. Configure the distributor with its TVS token via `setToken()`/`constructor`.
2. Create a vesting allocation whose `Allocation.token` is also the TVS token (the intended design).
3. Call `getUnclaimedAmounts(nftId)`.
4. The check evaluates to **true** because both refer to the same TVS token, therefore, the function immediately returns `0`, skipping all vesting‑flow logic.

### Impact

- All NFTs holding the TVS token will have `getUnclaimedAmounts` **always return 0**, so legitimate unclaimed dividends are never credited.
- Conversely, NFTs holding **other tokens** will bypass this faulty check and **incorrectly contribute to dividend calculations**, inflating their share.

### PoC

I created a standalone PoC that isolates the `getTotalUnclaimedAmounts` logic, because the full system has multiple interacting bugs that prevent reliably reaching this code path in its deployed form.

Add the following test suite into `test/getUnclaimedAmounts.t.sol` and run the test

```solidity
forge clean
forge test --mt test_POC_SameToken_AlwaysZero -vvvv
```

You will see that it will return zero.

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
            address(TOKEN), // same token, which is wanted
            amounts,
            secondsClaimed,
            periods,
            claimedFlows
        );

        vm.expectRevert(); // function never increments i → infinite loop → OOG revert
        poc.getUnclaimedAmounts(1);
    }

    function test_POC_SameToken_AlwaysZero() public {
        // Create a valid unclaimed vesting flow
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 5000;

        uint256[] memory secondsClaimed = new uint256[](1);
        secondsClaimed[0] = 0;

        uint256[] memory periods = new uint256[](1);
        periods[0] = 100;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        // Vesting token == distributor token → triggers bug
        vesting.setAllocation(
            1,
            TOKEN, // SAME TOKEN → function returns zero
            amounts,
            secondsClaimed,
            periods,
            claimedFlows
        );

        // Expect zero despite unclaimed amounts existing
        uint256 result = poc.getUnclaimedAmounts(1);

        assertEq(result, 0, "BUG: function should not return zero for same token");
    }
}
```

### Mitigation

Switch the logic to return 0 if the tokens do not match

```diff
-if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  