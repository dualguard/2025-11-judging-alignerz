# [000697] Array Length Mismatch in Distribution Functions Causes DoS
  
  ### Summary

Missing array length validation in `distributeRewardTVS()` and `distributeStablecoinAllocation()` will cause a Denial of Service for KOL reward distribution as the functions iterate over the storage array length but access the input parameter array without validation, causing out-of-bounds reverts when array lengths mismatch.

### Root Cause

In [AlignerzVesting.sol:525-535](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535) and [AlignerzVesting.sol:540-550](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550), there is a missing validation that the input `kol` array length matches the storage array length (`kolTVSAddresses.length` or `kolStablecoinAddresses.length`).

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;  // @audit Storage array length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);  // @audit Accesses input array without length check
        unchecked {
            ++i;
        }
    }
}

function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolStablecoinAddresses.length;  // @audit Storage array length
    for (uint256 i; i < len;) {
        _claimStablecoinAllocation(rewardProjectId, kol[i]);  // @audit Accesses input array without length check
        unchecked {
            ++i;
        }
    }
}
```

The loop iterates `len` times (based on storage array length), but accesses `kol[i]` from the input parameter. If `kol.length < len`, the function reverts with an out-of-bounds array access error.

### Internal Pre-conditions

1. Admin needs to call `launchRewardProject()` and `setTVSAllocation()` or `setStablecoinAllocation()` to set up KOL rewards
2. The claim deadline needs to pass (i.e., `block.timestamp > rewardProject.claimDeadline`)
3. At least one KOL needs to have not claimed their allocation (so `kolTVSAddresses.length > 0` or `kolStablecoinAddresses.length > 0`)


### External Pre-conditions

None required.


### Attack Path

This is a vulnerability path rather than an attack path:

1. Admin sets up a reward project with 5 KOLs allocated TVS rewards via `setTVSAllocation()`
2. The claim deadline passes without all KOLs claiming their rewards
3. Admin calls `distributeRewardTVS()` with an array of only 3 addresses (incorrect length - should be 5)
4. The function sets `len = 5` (storage array length) and loops 5 times
5. On iteration 4, accessing `kol[3]` causes an out-of-bounds revert since the input array only has 3 elements
6. The transaction reverts and rewards cannot be distributed


### Impact

The KOLs who have not claimed their rewards before the deadline cannot receive their allocated tokens through the distribution mechanism. The protocol suffers operational failure as the admin cannot execute the intended reward distribution functionality.

While `distributeRemainingRewardTVS()` and `distributeRemainingStablecoinAllocation()` exist as alternative functions that iterate over the storage array directly, the primary distribution functions `distributeRewardTVS()` and `distributeStablecoinAllocation()` are rendered non-functional due to this bug.

### PoC

Save this POC to `protocol/test/H01_ArrayLengthMismatchDistribution.t.sol`

Run with:
```bash
forge test --match-contract H01_ArrayLengthMismatchDistributionTest -vvv
```

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

/// @title H-01: Array Length Mismatch in Distribution Functions POC
/// @notice Demonstrates how distributeRewardTVS and distributeStablecoinAllocation
///         iterate over storage array length but access input array without validation,
///         causing out-of-bounds reverts and DoS when arrays have mismatched lengths.
contract H01_ArrayLengthMismatchDistributionTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public stablecoin;

    address public owner;
    address public treasury;

    address public kol1;
    address public kol2;
    address public kol3;
    address public kol4;
    address public kol5;

    uint256 constant TOKEN_AMOUNT = 10_000_000 ether;
    uint256 constant STABLECOIN_AMOUNT = 100_000 * 1e6;
    uint256 constant REWARD_PROJECT_ID = 0;
    uint256 constant VESTING_PERIOD = 365 days;
    uint256 constant CLAIM_WINDOW = 7 days;

    function setUp() public {
        owner = address(this);
        treasury = makeAddr("treasury");

        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");
        kol3 = makeAddr("kol3");
        kol4 = makeAddr("kol4");
        kol5 = makeAddr("kol5");

        token = new Aligners26("26Aligners", "A26Z");
        stablecoin = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(treasury);

        token.approve(address(vesting), type(uint256).max);
        stablecoin.approve(address(vesting), type(uint256).max);
        stablecoin.mint(address(this), STABLECOIN_AMOUNT);
    }

    /// @notice Demonstrates DoS in distributeRewardTVS when providing fewer KOL addresses
    ///         than the storage array length
    function test_H01_DistributeRewardTVS_ArrayLengthMismatch_RevertOnFewerAddresses() public {
        address[] memory kolAddresses = new address[](5);
        kolAddresses[0] = kol1;
        kolAddresses[1] = kol2;
        kolAddresses[2] = kol3;
        kolAddresses[3] = kol4;
        kolAddresses[4] = kol5;

        uint256[] memory tvsAmounts = new uint256[](5);
        tvsAmounts[0] = 1000 ether;
        tvsAmounts[1] = 2000 ether;
        tvsAmounts[2] = 3000 ether;
        tvsAmounts[3] = 4000 ether;
        tvsAmounts[4] = 5000 ether;

        uint256 totalTVSAllocation = 15000 ether;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            CLAIM_WINDOW
        );

        vesting.setTVSAllocation(
            REWARD_PROJECT_ID,
            totalTVSAllocation,
            VESTING_PERIOD,
            kolAddresses,
            tvsAmounts
        );

        vm.warp(block.timestamp + CLAIM_WINDOW + 1);

        // Providing only 3 addresses when storage array has 5 KOLs
        // The loop iterates 5 times (storage array length) but input array only has 3 elements
        address[] memory partialKolList = new address[](3);
        partialKolList[0] = kol1;
        partialKolList[1] = kol2;
        partialKolList[2] = kol3;

        // This reverts with out-of-bounds array access when trying to access kol[3] and kol[4]
        vm.expectRevert();
        vesting.distributeRewardTVS(REWARD_PROJECT_ID, partialKolList);
    }

    /// @notice Demonstrates DoS in distributeStablecoinAllocation with mismatched array length
    function test_H01_DistributeStablecoinAllocation_ArrayLengthMismatch_RevertOnFewerAddresses() public {
        address[] memory kolAddresses = new address[](5);
        kolAddresses[0] = kol1;
        kolAddresses[1] = kol2;
        kolAddresses[2] = kol3;
        kolAddresses[3] = kol4;
        kolAddresses[4] = kol5;

        uint256[] memory stablecoinAmounts = new uint256[](5);
        stablecoinAmounts[0] = 10000 * 1e6;
        stablecoinAmounts[1] = 15000 * 1e6;
        stablecoinAmounts[2] = 20000 * 1e6;
        stablecoinAmounts[3] = 25000 * 1e6;
        stablecoinAmounts[4] = 30000 * 1e6;

        uint256 totalStablecoinAllocation = 100000 * 1e6;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            CLAIM_WINDOW
        );

        vesting.setStablecoinAllocation(
            REWARD_PROJECT_ID,
            totalStablecoinAllocation,
            kolAddresses,
            stablecoinAmounts
        );

        vm.warp(block.timestamp + CLAIM_WINDOW + 1);

        // Providing only 2 addresses when storage array has 5 KOLs
        address[] memory partialKolList = new address[](2);
        partialKolList[0] = kol1;
        partialKolList[1] = kol2;

        vm.expectRevert();
        vesting.distributeStablecoinAllocation(REWARD_PROJECT_ID, partialKolList);
    }

    /// @notice Demonstrates that after some KOLs claim, the remaining distribution still fails
    ///         if the provided array doesn't match the remaining storage array length
    function test_H01_DistributeRewardTVS_AfterPartialClaims_StillMismatched() public {
        address[] memory kolAddresses = new address[](5);
        kolAddresses[0] = kol1;
        kolAddresses[1] = kol2;
        kolAddresses[2] = kol3;
        kolAddresses[3] = kol4;
        kolAddresses[4] = kol5;

        uint256[] memory tvsAmounts = new uint256[](5);
        tvsAmounts[0] = 1000 ether;
        tvsAmounts[1] = 2000 ether;
        tvsAmounts[2] = 3000 ether;
        tvsAmounts[3] = 4000 ether;
        tvsAmounts[4] = 5000 ether;

        uint256 totalTVSAllocation = 15000 ether;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            CLAIM_WINDOW
        );

        vesting.setTVSAllocation(
            REWARD_PROJECT_ID,
            totalTVSAllocation,
            VESTING_PERIOD,
            kolAddresses,
            tvsAmounts
        );

        // KOL1 and KOL2 claim their TVS before deadline
        vm.prank(kol1);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);

        vm.prank(kol2);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);

        // After claims, storage array has 3 KOLs remaining (kol3, kol4, kol5)
        // But due to swap-and-pop, the order might be different

        vm.warp(block.timestamp + CLAIM_WINDOW + 1);

        // Providing only 2 addresses when storage array has 3 KOLs remaining
        address[] memory wrongLengthList = new address[](2);
        wrongLengthList[0] = kol3;
        wrongLengthList[1] = kol4;

        vm.expectRevert();
        vesting.distributeRewardTVS(REWARD_PROJECT_ID, wrongLengthList);
    }

    /// @notice Demonstrates the full impact: owner cannot distribute rewards at all
    ///         when they don't have the exact array length match
    function test_H01_FullImpact_OwnerCannotDistributeWithWrongArrayLength() public {
        address[] memory kolAddresses = new address[](3);
        kolAddresses[0] = kol1;
        kolAddresses[1] = kol2;
        kolAddresses[2] = kol3;

        uint256[] memory tvsAmounts = new uint256[](3);
        tvsAmounts[0] = 1000 ether;
        tvsAmounts[1] = 2000 ether;
        tvsAmounts[2] = 3000 ether;

        uint256 totalTVSAllocation = 6000 ether;

        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            CLAIM_WINDOW
        );

        vesting.setTVSAllocation(
            REWARD_PROJECT_ID,
            totalTVSAllocation,
            VESTING_PERIOD,
            kolAddresses,
            tvsAmounts
        );

        vm.warp(block.timestamp + CLAIM_WINDOW + 1);

        uint256 kol1BalanceBefore = token.balanceOf(kol1);
        uint256 kol2BalanceBefore = token.balanceOf(kol2);
        uint256 kol3BalanceBefore = token.balanceOf(kol3);

        // Attempt 1: Providing empty array - REVERTS
        address[] memory emptyList = new address[](0);
        vm.expectRevert();
        vesting.distributeRewardTVS(REWARD_PROJECT_ID, emptyList);

        // Attempt 2: Providing only 1 address - REVERTS
        address[] memory oneAddressList = new address[](1);
        oneAddressList[0] = kol1;
        vm.expectRevert();
        vesting.distributeRewardTVS(REWARD_PROJECT_ID, oneAddressList);

        // Attempt 3: Providing only 2 addresses - REVERTS
        address[] memory twoAddressList = new address[](2);
        twoAddressList[0] = kol1;
        twoAddressList[1] = kol2;
        vm.expectRevert();
        vesting.distributeRewardTVS(REWARD_PROJECT_ID, twoAddressList);

        // Verify no tokens were distributed
        assertEq(token.balanceOf(kol1), kol1BalanceBefore, "KOL1 should not have received tokens");
        assertEq(token.balanceOf(kol2), kol2BalanceBefore, "KOL2 should not have received tokens");
        assertEq(token.balanceOf(kol3), kol3BalanceBefore, "KOL3 should not have received tokens");
    }
}

```

### Mitigation

Add array length validation before the loop in both functions:

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    require(kol.length == len, Array_Lengths_Must_Match());  // Add validation
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}

function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolStablecoinAddresses.length;
    require(kol.length == len, Array_Lengths_Must_Match());  // Add validation
    for (uint256 i; i < len;) {
        _claimStablecoinAllocation(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}
```

  