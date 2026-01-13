# [000696] Division by Zero in `getClaimableAmountAndSeconds` Permanently Locks KOL Tokens
  
  ### Summary

Missing validation for `vestingPeriod > 0` in `setTVSAllocation()` will cause permanent denial of service for KOLs as admin setting `vestingPeriod = 0` will result in division by zero when KOLs attempt to claim their vested tokens.

### Root Cause

In [AlignerzVesting.sol:458-460](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L458-L460) the `setTVSAllocation()` function accepts `vestingPeriod` parameter without validating it is greater than zero:

```solidity
function setTVSAllocation(uint256 rewardProjectId, uint256 totalTVSAllocation, uint256 vestingPeriod, address[] calldata kolTVS, uint256[] calldata TVSamounts) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    rewardProject.vestingPeriod = vestingPeriod;  // @audit No validation for vestingPeriod > 0
    // ...
}
```

This zero value propagates to allocations in [AlignerzVesting.sol:608-609](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L608-L609):

```solidity
uint256 vestingPeriod = rewardProject.vestingPeriod;
rewardProject.allocations[nftId].vestingPeriods.push(vestingPeriod);
```

When KOLs call `claimTokens()`, the function `getClaimableAmountAndSeconds()` at [AlignerzVesting.sol:993](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L993) performs division by `vestingPeriod`:

```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;  // @audit Division by zero if vestingPeriod == 0
```

Note: Bidding projects are protected via `require(vestingPeriod > 0, Zero_Value())` in `placeBid()` at line 722, but reward projects lack this validation.


### Internal Pre-conditions

1. Admin needs to call `launchRewardProject()` to create a new reward project
2. Admin needs to call `setTVSAllocation()` to set `vestingPeriod` to exactly `0` (accidentally or intentionally)
3. KOL needs to call `claimRewardTVS()` to receive their NFT allocation with `vestingPeriod = 0`


### External Pre-conditions

None

### Attack Path

1. Admin calls `launchRewardProject()` to create a reward project for KOLs
2. Admin calls `setTVSAllocation()` with `vestingPeriod = 0` (mistakenly entering 0 instead of intended vesting duration)
3. KOL calls `claimRewardTVS()` which mints an NFT and creates an allocation with `vestingPeriods[0] = 0`
4. Time passes and KOL calls `claimTokens()` to claim their vested tokens
5. Inside `claimTokens()`, the function calls `getClaimableAmountAndSeconds()`
6. The calculation `claimableAmount = (amount * claimableSeconds) / vestingPeriod` executes with `vestingPeriod = 0`
7. Transaction reverts with panic code 0x12 (division by zero)
8. KOL can never claim their tokens - they are permanently locked


### Impact

The KOLs suffer a 100% loss of their allocated tokens. All tokens allocated to the affected reward project become permanently locked in the contract with no recovery mechanism. The severity increases with the number of KOLs affected - if 10 KOLs each have 10,000 tokens allocated, 100,000 tokens are permanently lost.

Additionally:
- The `claimTokens()` function becomes permanently unusable for all affected allocations
- There is no admin function to modify existing allocations or rescue tokens
- The NFTs remain in KOLs' wallets but are worthless as they can never be redeemed

### PoC

Save this POC to `protocol/test/H02_DivisionByZero.t.sol`

Run with: `forge test --match-contract H02_DivisionByZeroTest -vvv`

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract H02_DivisionByZeroTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public kol;
    address public treasury;

    uint256 constant TOKEN_AMOUNT = 1_000_000 ether;
    uint256 constant KOL_ALLOCATION = 10_000 ether;

    function setUp() public {
        owner = address(this);
        kol = makeAddr("kol");
        treasury = makeAddr("treasury");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(treasury);

        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_DivisionByZero_RewardProject_ZeroVestingPeriod() public {
        uint256 rewardProjectId = 0;
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = KOL_ALLOCATION;

        vesting.setTVSAllocation(
            rewardProjectId,
            KOL_ALLOCATION,
            0,
            kols,
            amounts
        );

        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);

        uint256 nftId = 1;
        assertEq(nft.ownerOf(nftId), kol);

        vm.warp(block.timestamp + 1 days);

        vm.prank(kol);
        vm.expectRevert();
        vesting.claimTokens(rewardProjectId, nftId);
    }

    function test_DivisionByZero_PermanentDoS() public {
        uint256 rewardProjectId = 0;
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 365 days;

        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        uint256 numKols = 5;
        address[] memory kols = new address[](numKols);
        uint256[] memory amounts = new uint256[](numKols);
        uint256 totalAllocation = 0;

        for (uint256 i = 0; i < numKols; i++) {
            kols[i] = makeAddr(string.concat("kol", vm.toString(i)));
            amounts[i] = 10_000 ether;
            totalAllocation += amounts[i];
        }

        token.approve(address(vesting), totalAllocation);

        vesting.setTVSAllocation(
            rewardProjectId,
            totalAllocation,
            0,
            kols,
            amounts
        );

        uint256[] memory nftIds = new uint256[](numKols);
        for (uint256 i = 0; i < numKols; i++) {
            vm.prank(kols[i]);
            vesting.claimRewardTVS(rewardProjectId);
            nftIds[i] = i + 1;
        }

        vm.warp(block.timestamp + 30 days);

        for (uint256 i = 0; i < numKols; i++) {
            vm.prank(kols[i]);
            vm.expectRevert();
            vesting.claimTokens(rewardProjectId, nftIds[i]);
        }

        vm.warp(block.timestamp + 365 days);

        for (uint256 i = 0; i < numKols; i++) {
            vm.prank(kols[i]);
            vm.expectRevert();
            vesting.claimTokens(rewardProjectId, nftIds[i]);
        }
    }

    function test_DivisionByZero_DirectFunctionCall() public {
        vm.warp(100 days);

        AlignerzVesting.Allocation memory allocation;
        allocation.amounts = new uint256[](1);
        allocation.amounts[0] = 1000 ether;
        allocation.vestingPeriods = new uint256[](1);
        allocation.vestingPeriods[0] = 0;
        allocation.vestingStartTimes = new uint256[](1);
        allocation.vestingStartTimes[0] = block.timestamp - 1 days;
        allocation.claimedSeconds = new uint256[](1);
        allocation.claimedSeconds[0] = 0;
        allocation.claimedFlows = new bool[](1);
        allocation.claimedFlows[0] = false;

        vm.expectRevert();
        vesting.getClaimableAmountAndSeconds(allocation, 0);
    }

    function test_NormalFlow_NonZeroVestingPeriod() public {
        uint256 rewardProjectId = 0;
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        uint256 vestingPeriod = 90 days;

        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = KOL_ALLOCATION;

        vesting.setTVSAllocation(
            rewardProjectId,
            KOL_ALLOCATION,
            vestingPeriod,
            kols,
            amounts
        );

        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);

        uint256 nftId = 1;

        vm.warp(block.timestamp + 30 days);

        uint256 balanceBefore = token.balanceOf(kol);
        vm.prank(kol);
        vesting.claimTokens(rewardProjectId, nftId);
        uint256 balanceAfter = token.balanceOf(kol);

        assertGt(balanceAfter, balanceBefore);
    }
}
```

### Mitigation

Add validation in `setTVSAllocation()` to ensure `vestingPeriod > 0`:

```solidity
function setTVSAllocation(
    uint256 rewardProjectId,
    uint256 totalTVSAllocation,
    uint256 vestingPeriod,
    address[] calldata kolTVS,
    uint256[] calldata TVSamounts
) external onlyOwner {
    require(vestingPeriod > 0, Zero_Value());  // Add this check

    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    rewardProject.vestingPeriod = vestingPeriod;
    // ... rest of function
}
```

Alternatively, add a zero check in `getClaimableAmountAndSeconds()`:

```solidity
function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
    // ...
    uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
    require(vestingPeriod > 0, "Vesting period cannot be zero");
    // ...
}
```

  