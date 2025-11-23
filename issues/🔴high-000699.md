# [000699] Integer Underflow in Distribution Loop Initialization Causes DoS
  
  ### Summary

In `distributeRemainingRewardTVS()` and `distributeRemainingStablecoinAllocation()`, the loop initialization `uint256 i = len - 1` will cause an arithmetic underflow when the array is empty (`len = 0`), resulting in a complete denial of service for the owner as any call to these functions will revert when all KOLs have already claimed or no KOLs were configured.


### Root Cause


- In [AlignerzVesting.sol:558](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L558) the loop initialization `for (uint256 i = len - 1; ...)` causes an arithmetic underflow when `len = 0`
- In [AlignerzVesting.sol:585](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L585) the same pattern exists for stablecoin distribution

The vulnerable code pattern:

```solidity
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {  // @audit underflow when len=0
        // ...
    }
}
```

In Solidity 0.8.29, when `len = 0`, the expression `len - 1` evaluates to `0 - 1` which causes a panic due to arithmetic underflow.

### Internal Pre-conditions

1. Owner needs to call `launchRewardProject()` to create a reward project
2. One of the following scenarios must occur:
   - All KOLs claim their allocations before the deadline (emptying `kolTVSAddresses` or `kolStablecoinAddresses` arrays)
   - OR no KOL allocations are configured (arrays remain empty)
   - OR `setTVSAllocation()` / `setStablecoinAllocation()` is never called for the project

### External Pre-conditions

None required.

### Attack Path

This is a vulnerability path rather than an attack path:

1. Owner calls `launchRewardProject()` to create a new reward project with a claim deadline
2. Owner calls `setTVSAllocation()` to allocate TVS rewards to KOLs (e.g., 2 KOLs)
3. All KOLs call `claimRewardTVS()` before the deadline, which removes them from `kolTVSAddresses` array via `pop()`
4. After the claim deadline passes, owner calls `distributeRemainingRewardTVS()` to finalize the project
5. Since `kolTVSAddresses.length = 0`, the expression `len - 1` causes arithmetic underflow
6. Transaction reverts with panic code 0x11 (arithmetic underflow)

Alternative path:
1. Owner creates a reward project but never sets any KOL allocations
2. After deadline, owner attempts to call distribution functions
3. Same underflow occurs due to empty arrays

### Impact

The owner cannot execute `distributeRemainingRewardTVS()` or `distributeRemainingStablecoinAllocation()` when the respective KOL arrays are empty. This causes:

1. Denial of Service: Owner is blocked from finalizing reward project distributions
2. Operational Impact: The protocol cannot properly complete the lifecycle of reward projects where all participants have already claimed
3. State Inconsistency: Projects may remain in an incomplete state even though all rewards have been distributed

While no direct fund loss occurs (KOLs who claimed already received their rewards), this breaks expected protocol functionality and prevents clean project finalization.


### PoC

Save this POC to `protocol/test/C06_IntegerUnderflowDistributionLoop.t.sol`

Run with: `forge test --match-contract C06_IntegerUnderflowDistributionLoop -vvv`

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

contract C06_IntegerUnderflowDistributionLoop is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public stablecoin;

    address public owner;
    address public kol1;
    address public kol2;

    uint256 constant TOKEN_AMOUNT = 10_000 ether;
    uint256 constant STABLECOIN_AMOUNT = 5_000 ether;

    function setUp() public {
        owner = address(this);
        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");

        stablecoin = new MockUSD();
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
        vesting.setTreasury(address(1));
    }

    function test_POC_IntegerUnderflowWhenTVSArrayEmpty() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 100;
        uint256 projectId = 0;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            startTime,
            claimWindow
        );

        address[] memory kolTVS = new address[](2);
        kolTVS[0] = kol1;
        kolTVS[1] = kol2;

        uint256[] memory tvsAmounts = new uint256[](2);
        tvsAmounts[0] = 5_000 ether;
        tvsAmounts[1] = 5_000 ether;

        token.approve(address(vesting), TOKEN_AMOUNT);
        vesting.setTVSAllocation(projectId, TOKEN_AMOUNT, 30 days, kolTVS, tvsAmounts);

        vm.prank(kol1);
        vesting.claimRewardTVS(projectId);

        vm.prank(kol2);
        vesting.claimRewardTVS(projectId);

        vm.warp(startTime + claimWindow + 1);

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingRewardTVS(projectId);
    }

    function test_POC_IntegerUnderflowWhenStablecoinArrayEmpty() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 100;
        uint256 projectId = 0;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            startTime,
            claimWindow
        );

        address[] memory kolStablecoin = new address[](2);
        kolStablecoin[0] = kol1;
        kolStablecoin[1] = kol2;

        uint256[] memory stablecoinAmounts = new uint256[](2);
        stablecoinAmounts[0] = 2_500 ether;
        stablecoinAmounts[1] = 2_500 ether;

        stablecoin.mint(owner, STABLECOIN_AMOUNT);
        stablecoin.approve(address(vesting), STABLECOIN_AMOUNT);
        vesting.setStablecoinAllocation(projectId, STABLECOIN_AMOUNT, kolStablecoin, stablecoinAmounts);

        vm.prank(kol1);
        vesting.claimStablecoinAllocation(projectId);

        vm.prank(kol2);
        vesting.claimStablecoinAllocation(projectId);

        vm.warp(startTime + claimWindow + 1);

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingStablecoinAllocation(projectId);
    }

    function test_POC_IntegerUnderflowWithNoKOLsSetup() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 100;
        uint256 projectId = 0;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            startTime,
            claimWindow
        );

        vm.warp(startTime + claimWindow + 1);

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingRewardTVS(projectId);

        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingStablecoinAllocation(projectId);
    }
}

```

### Mitigation

Add an early return check when the array is empty:

```solidity
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;

    if (len == 0) return;  // Early return if nothing to distribute

    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
        // ... existing logic ...
    }
}

function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolStablecoinAddresses.length;

    if (len == 0) return;  // Early return if nothing to distribute

    for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
        // ... existing logic ...
    }
}
```

  