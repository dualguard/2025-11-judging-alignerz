# [000416] Mixed-start vesting flows revert claims when any flow isn’t yet claimable
  
  https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L960-L995 – `claimTokens` iterates all unclaimed flows and `getClaimableAmountAndSeconds` requires `claimableAmount > 0` for each. If any flow has zero claimable (e.g., its vesting hasn’t started yet or it’s already up‑to‑date), the entire call reverts, so already‑vested flows can’t be withdrawn.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1046 – `mergeTVS` lets you merge arbitrary `projectIds` as long as the token matches, and it appends each flow’s `vestingStartTimes`/`claimedSeconds` without aligning them. 

Merging flows from projects with different start times (or a reward TVS whose `startTime` is in the future) creates an NFT where some flows are unclaimable. Calling `claimTokens` before the latest start time reverts (and can even underflow at https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L989), locking vested tokens until every flow is claimable. No invariant enforces synchronized vesting, so the DoS is reachable.

## Impact
I’d rate this High severity because the bug can permanently block token withdrawals for an NFT (severe availability loss and funds unusable) when flows have different start times, and it’s easy to hit via the public `mergeTVS` path or simply by merging projects with differing start times (It has a high likelihood and no exotic preconditions).

## POC
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {Options} from "openzeppelin-foundry-upgrades/Options.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

// PoC: merging flows with different start times makes claimTokens revert until every flow has started.
contract AlignerzVestingMixedStartPoC is Test {
    AlignerzVesting private vesting;
    Aligners26 private token;
    AlignerzNFT private nft;
    MockUSD private usdt;

    // RewardProject base slot = 112 (from storage layout).
    uint256 private constant REWARD_PROJECTS_SLOT = 112;

    address private alice;
    uint256 private startTime;

    function setUp() public {
        alice = makeAddr("alice");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        Options memory opts;
        opts.unsafeSkipAllChecks = true;

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft))),
                opts
            )
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Allow vesting contract to pull all tokens needed for allocations.
        token.approve(address(vesting), type(uint256).max);
        startTime = block.timestamp;
    }

    function test_MergeDifferentStartTimesBlocksClaims() public {
        // Reward project #0: starts now.
        address[] memory recipients = new address[](1);
        recipients[0] = alice;

        uint256[] memory amountsNow = new uint256[](1);
        amountsNow[0] = 100 ether;
        uint256 vestingPeriod = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, 60 days);
        vesting.setTVSAllocation(0, amountsNow[0], vestingPeriod, recipients, amountsNow);

        vm.prank(alice);
        vesting.claimRewardTVS(0);
        uint256 nftNow = nft.getTotalMinted();

        // Reward project #1: starts in the future.
        uint256 futureStart = startTime + 10 days;
        uint256[] memory amountsFuture = new uint256[](1);
        amountsFuture[0] = 50 ether;

        vesting.launchRewardProject(address(token), address(usdt), futureStart, 60 days);
        vesting.setTVSAllocation(1, amountsFuture[0], vestingPeriod, recipients, amountsFuture);

        vm.prank(alice);
        vesting.claimRewardTVS(1);

        // Manually attach the future-start flow to nftNow to mimic a merged TVS with mixed start times.
        _overrideAllocationWithTwoFlows({
            projectId: 0,
            nftId: nftNow,
            amount0: amountsNow[0],
            amount1: amountsFuture[0],
            vestingPeriod0: vestingPeriod,
            vestingPeriod1: vestingPeriod,
            startTime0: startTime,
            startTime1: futureStart
        });

        // Advance time: first flow is partially vested, second flow has not started yet.
        vm.warp(startTime + 8 days);

        // Expect revert: claimTokens requires every flow to be claimable, so the pending future flow blocks all claims.
        vm.startPrank(alice);
        vm.expectRevert();
        vesting.claimTokens(0, nftNow);
        vm.stopPrank();
    }

    // Writes a 2-flow allocation directly into storage to avoid the broken mergeTVS/splitTVS helpers.
    function _overrideAllocationWithTwoFlows(
        uint256 projectId,
        uint256 nftId,
        uint256 amount0,
        uint256 amount1,
        uint256 vestingPeriod0,
        uint256 vestingPeriod1,
        uint256 startTime0,
        uint256 startTime1
    ) internal {
        bytes32 amountsSlot = _allocationFieldSlot(projectId, nftId, 0);
        bytes32 vestingPeriodsSlot = _allocationFieldSlot(projectId, nftId, 1);
        bytes32 vestingStartSlot = _allocationFieldSlot(projectId, nftId, 2);
        bytes32 claimedSecondsSlot = _allocationFieldSlot(projectId, nftId, 3);
        bytes32 claimedFlowsSlot = _allocationFieldSlot(projectId, nftId, 4);

        uint256[] memory two = new uint256[](2);

        two[0] = amount0;
        two[1] = amount1;
        _writeUintArray(amountsSlot, two);

        two[0] = vestingPeriod0;
        two[1] = vestingPeriod1;
        _writeUintArray(vestingPeriodsSlot, two);

        two[0] = startTime0;
        two[1] = startTime1;
        _writeUintArray(vestingStartSlot, two);

        two[0] = 0;
        two[1] = 0;
        _writeUintArray(claimedSecondsSlot, two);

        _writeBoolArray(claimedFlowsSlot, 2);
    }

    function _allocationFieldSlot(uint256 projectId, uint256 nftId, uint256 fieldOffset) internal pure returns (bytes32) {
        // rewardProjects mapping slot = 112; allocations is field #3 inside RewardProject.
        bytes32 projectSlot = keccak256(abi.encode(projectId, REWARD_PROJECTS_SLOT));
        bytes32 allocationsMappingSlot = bytes32(uint256(projectSlot) + 3);
        bytes32 allocationBase = keccak256(abi.encode(nftId, allocationsMappingSlot));
        return bytes32(uint256(allocationBase) + fieldOffset);
    }

    function _writeUintArray(bytes32 slot, uint256[] memory values) internal {
        vm.store(address(vesting), slot, bytes32(values.length));
        bytes32 dataSlot = keccak256(abi.encode(slot));
        for (uint256 i; i < values.length; i++) {
            vm.store(address(vesting), bytes32(uint256(dataSlot) + i), bytes32(values[i]));
        }
    }

    function _writeBoolArray(bytes32 slot, uint256 length) internal {
        vm.store(address(vesting), slot, bytes32(length));
        bytes32 dataSlot = keccak256(abi.encode(slot));
        // Reset first 32 flags to 0 (enough for length=2).
        vm.store(address(vesting), dataSlot, bytes32(0));
    }
}

```


## Recommendation
In `claimTokens`/`getClaimableAmountAndSeconds`,  treat non-vested flows as `claimableAmount = 0` instead of reverting. Guard `if (block.timestamp <= vestingStartTimes[i]) return (0, 0);` and drop the `require(claimableAmount > 0);` in the loop, `if (claimableAmount == 0) continue;` so already-vested flows can be withdrawn.
  