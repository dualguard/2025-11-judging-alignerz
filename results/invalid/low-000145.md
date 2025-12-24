# [000145] Owner sweep reverts when no KOLs remain to distribute
  
  ### Summary

Missing empty-array guard in the post-deadline sweep functions causes an underflow when no `KOL`s are pending, so the owner cannot call the cleanup helpers if everyone already claimed.

### Root Cause

In AlignerzVesting.sol, both `distributeRemainingRewardTVS` and `distributeRemainingStablecoinAllocation` do `uint256 i = len - 1;` without handling `len == 0`, so an empty `kolTVSAddresses/kolStablecoinAddresses` array triggers an underflow revert instead of a no-op.

### Internal Pre-conditions

1. A reward project exists and its `kolTVSAddresses` (or `kolStablecoinAddresses`) array is empty (e.g., all KOLs claimed or none were allocated).
2. The caller is the owner, invoking the sweep after `claimDeadline`.

### External Pre-conditions

None

### Attack Path

Failure Path:
1. Owner calls `distributeRemainingRewardTVS(rewardProjectId)` or `distributeRemainingStablecoinAllocation(rewardProjectId)` after `claimDeadline`.
2. `len` is zero; `uint256 i = len - 1;` underflows; the call reverts with arithmetic error.

### Impact

Owner cannot run the post-deadline sweep when there is nothing left to distribute (functional DoS of the helper; no funds at risk)

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";

/// @dev PoC: post-deadline distributions underflow when no KOLs remain (len=0 => len-1 panics).
contract RemainingDistributionBugTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    MockUSD stable;

    function setUp() public {
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri/");
        token = new Aligners26("Aligners", "ALN");
        stable = new MockUSD();

        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Launch reward project but do not allocate any KOLs, so arrays stay length 0.
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 1 days);
    }

    function test_DistributeRemainingRewardTVS_RevertsWhenNoKOLs() public {
        vm.warp(block.timestamp + 2 days); // Past claimDeadline

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingRewardTVS(0);
    }

    function test_DistributeRemainingStablecoin_RevertsWhenNoKOLs() public {
        vm.warp(block.timestamp + 2 days); // Past claimDeadline

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingStablecoinAllocation(0);
    }
}
```

### Mitigation

Add an early return if `len == 0` before computing `len - 1`, or rewrite the loops to handle empty arrays safely.
  