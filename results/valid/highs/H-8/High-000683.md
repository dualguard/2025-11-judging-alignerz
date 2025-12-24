# [000683] [H-2] Dividend distributor infinite loop bricks payouts
  
  ### Summary

The dividend distributor’s `getUnclaimedAmounts` loop never increments its index when a flow is already claimed or unclaimed seconds are zero, causing an infinite loop and out-of-gas; this bricks dividend setup and payouts, trapping deposited stablecoins.

### Root Cause

In `A26ZDividendDistributor.sol:147-159`, the `for (uint i; i < len;)` loop `continue`s for `claimedFlows[i]` and `claimedSeconds[i] == 0` without incrementing `i`, so `i` remains constant and the loop never progresses.

### Internal Pre-conditions

  1. At least one NFT allocation exists (normal operation).
  2. For any flow where `claimedFlows[i]` is true, or for a fresh flow where `claimedSeconds[i] == 0`, the loop will hit `continue`.

### External Pre-conditions

None.

### Attack Path

  1. Owner or anyone calls `setUpTheDividends`/`setDividends`/`setAmounts` (all rely on `getUnclaimedAmounts`).
  2. Function enters `getUnclaimedAmounts`; hits a flow with `claimedFlows[i] == true` or `claimedSeconds[i] == 0` (fresh allocation).
  3. Loop `continue`s without incrementing `i`, repeats forever, and runs out of gas.

### Impact

Dividend configuration and distribution are impossible; calls OOG, leaving any deposited stablecoin permanently stuck in the distributor. TVS holders cannot receive dividends.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";

contract DividendDistributorTest is Test {
    A26ZDividendDistributor private dist;
    MockVesting private vesting;
    MockNFT private nft;
    MockUSD private stable;

    function setUp() public {
        stable = new MockUSD();
        vesting = new MockVesting();
        nft = new MockNFT();
        // Use a different token address than the vesting allocation token to avoid the early return
        dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            30 days,
            address(0xCAFE)
        );
    }

    function test_getUnclaimedAmounts_outOfGasLoopFailsCall() public {
        // Limit gas so the non-incrementing loop runs out and the call returns false
        bytes memory callData = abi.encodeWithSelector(dist.getUnclaimedAmounts.selector, 0);
        (bool success,) = address(dist).call{gas: 50_000}(callData);
        assertFalse(success, "call should fail due to infinite loop/out-of-gas");
    }
}

contract MockVesting is IAlignerzVesting {
    Allocation private alloc;

    constructor() {
        alloc.amounts.push(1e18);
        alloc.vestingPeriods.push(30 days);
        alloc.vestingStartTimes.push(block.timestamp);
        alloc.claimedSeconds.push(0); // triggers the non-incrementing continue branch
        alloc.claimedFlows.push(false);
        alloc.token = IERC20(address(0xBEEF));
    }

    function allocationOf(uint256) external view returns (Allocation memory) {
        return alloc;
    }
}

contract MockNFT {
    function getTotalMinted() external pure returns (uint256) {
        return 1;
    }

    function extOwnerOf(uint256) external pure returns (address) {
        return address(1);
    }
}
```

### Mitigation

_No response_
  