# [000695] Array Index Manipulation Bug in `_claimRewardTVS` Causes State Corruption with Stale Out-of-Bounds Indices
  
  ### Summary

The buggy swap-and-pop logic in `_claimRewardTVS()` that sets `kolTVSIndexOf[kol] = arrayLength - 1` before the pop and never deletes the mapping will cause state corruption for the protocol as any KOL claiming their reward TVS will leave stale index mappings pointing to out-of-bounds array positions.

### Root Cause


In [AlignerzVesting.sol:615-621](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L615-L621), the swap-and-pop logic has two bugs:

1. Line 617 sets `kolTVSIndexOf[kol] = arrayLength - 1` before the array is popped, causing the removed user's index to point outside the array bounds after removal
2. The index mapping is never deleted after the user is removed from the array

```solidity
uint256 index = rewardProject.kolTVSIndexOf[kol];
uint256 arrayLength = rewardProject.kolTVSAddresses.length;
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;  // @audit BUG: Sets to last index BEFORE pop
address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
rewardProject.kolTVSAddresses.pop();
// @audit BUG: kolTVSIndexOf[kol] is never deleted
```

The same bug exists in `_claimStablecoinAllocation()` at lines 634-640.

### Internal Pre-conditions

1. Owner needs to call `launchRewardProject()` to create a reward project
2. Owner needs to call `setTVSAllocation()` to set KOL TVS allocations with at least 1 KOL


### External Pre-conditions

None

### Attack Path

1. Owner creates a reward project with 3 KOLs: `[Alice, Bob, Charlie]` with indices `{Alice: 0, Bob: 1, Charlie: 2}`
2. Alice calls `claimRewardTVS()` to claim her reward
3. The swap-and-pop logic executes:
   - `index = 0` (Alice's position)
   - `arrayLength = 3`
   - `kolTVSIndexOf[Alice] = 3 - 1 = 2` (Alice's index is now 2)
   - Charlie is swapped to position 0
   - Array becomes `[Charlie, Bob]` (length = 2)
4. Alice's stale index (2) now points outside the array bounds (valid indices are 0 and 1)
5. Any code that tries to access `kolTVSAddresses[kolTVSIndexOf[Alice]]` will revert with out-of-bounds error


### Impact

The protocol suffers state corruption where:

1. Out-of-bounds indices: Claimed users retain index mappings that point outside the array bounds, causing reverts if accessed
2. Index mappings never cleaned: Storage is not properly cleaned, leaving stale data that could confuse off-chain integrations or future protocol upgrades
3. Potential DoS: Any external integration or future protocol functionality that relies on `kolTVSIndexOf` mappings being accurate will fail or behave incorrectly

While the current codebase does not directly use the stale indices after claim (the `kolTVSRewards` check prevents double-claiming), this represents:
- Improper state management that violates expected invariants
- Potential attack surface for future code additions
- Storage bloat from never-deleted mappings

### PoC

Save this POC test in `protocol/test/H03_ArrayIndexManipulationBug.t.sol`

Run with:
```bash
forge test --match-path test/H03_ArrayIndexManipulationBug.t.sol -vvv
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
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract H03_AlignerzVestingHarness is AlignerzVesting {
    function getKolTVSIndexOf(uint256 projectId, address kol) external view returns (uint256) {
        return rewardProjects[projectId].kolTVSIndexOf[kol];
    }

    function getKolTVSAddressesLength(uint256 projectId) external view returns (uint256) {
        return rewardProjects[projectId].kolTVSAddresses.length;
    }

    function getKolTVSAddressAt(uint256 projectId, uint256 index) external view returns (address) {
        return rewardProjects[projectId].kolTVSAddresses[index];
    }

    function getKolTVSRewards(uint256 projectId, address kol) external view returns (uint256) {
        return rewardProjects[projectId].kolTVSRewards[kol];
    }
}

contract H03_ArrayIndexManipulationBugTest is Test {
    H03_AlignerzVestingHarness public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public alice;
    address public bob;
    address public charlie;

    uint256 constant TOKEN_AMOUNT = 10_000_000 ether;
    uint256 constant ALLOCATION_AMOUNT = 1000 ether;
    uint256 constant VESTING_PERIOD = 365 days;
    uint256 constant PROJECT_ID = 0;

    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        charlie = makeAddr("charlie");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        H03_AlignerzVestingHarness impl = new H03_AlignerzVestingHarness();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(impl),
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        );
        vesting = H03_AlignerzVestingHarness(payable(address(proxy)));

        nft.addMinter(address(proxy));
        vesting.setTreasury(address(1));

        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_H03_ArrayIndexManipulationBug_StaleIndexAfterClaim() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](3);
        kols[0] = alice;
        kols[1] = bob;
        kols[2] = charlie;

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = ALLOCATION_AMOUNT;
        amounts[1] = ALLOCATION_AMOUNT;
        amounts[2] = ALLOCATION_AMOUNT;

        uint256 totalAllocation = ALLOCATION_AMOUNT * 3;
        vesting.setTVSAllocation(PROJECT_ID, totalAllocation, VESTING_PERIOD, kols, amounts);

        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 3);
        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, alice), 0);
        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, bob), 1);
        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, charlie), 2);
        assertEq(vesting.getKolTVSAddressAt(PROJECT_ID, 0), alice);
        assertEq(vesting.getKolTVSAddressAt(PROJECT_ID, 1), bob);
        assertEq(vesting.getKolTVSAddressAt(PROJECT_ID, 2), charlie);

        vm.prank(alice);
        vesting.claimRewardTVS(PROJECT_ID);

        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 2);

        uint256 aliceStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, alice);

        assertEq(aliceStaleIndex, 2);

        assertGe(aliceStaleIndex, vesting.getKolTVSAddressesLength(PROJECT_ID));

        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, charlie), 0);
        assertEq(vesting.getKolTVSAddressAt(PROJECT_ID, 0), charlie);
        assertEq(vesting.getKolTVSAddressAt(PROJECT_ID, 1), bob);
    }

    function test_H03_ArrayIndexManipulationBug_MultipleClaimsCorruptState() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](3);
        kols[0] = alice;
        kols[1] = bob;
        kols[2] = charlie;

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = ALLOCATION_AMOUNT;
        amounts[1] = ALLOCATION_AMOUNT;
        amounts[2] = ALLOCATION_AMOUNT;

        uint256 totalAllocation = ALLOCATION_AMOUNT * 3;
        vesting.setTVSAllocation(PROJECT_ID, totalAllocation, VESTING_PERIOD, kols, amounts);

        vm.prank(alice);
        vesting.claimRewardTVS(PROJECT_ID);

        uint256 aliceStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, alice);
        assertEq(aliceStaleIndex, 2);
        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 2);

        vm.prank(bob);
        vesting.claimRewardTVS(PROJECT_ID);

        uint256 bobStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, bob);
        assertEq(bobStaleIndex, 1);
        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 1);
        assertGe(bobStaleIndex, vesting.getKolTVSAddressesLength(PROJECT_ID));

        vm.prank(charlie);
        vesting.claimRewardTVS(PROJECT_ID);

        uint256 charlieStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, charlie);
        assertEq(charlieStaleIndex, 0);
        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 0);

        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, alice), 2);
        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, bob), 1);
        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, charlie), 0);
    }

    function test_H03_ArrayIndexManipulationBug_IndexNotDeleted() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](1);
        kols[0] = alice;

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = ALLOCATION_AMOUNT;

        vesting.setTVSAllocation(PROJECT_ID, ALLOCATION_AMOUNT, VESTING_PERIOD, kols, amounts);

        assertEq(vesting.getKolTVSIndexOf(PROJECT_ID, alice), 0);
        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 1);

        vm.prank(alice);
        vesting.claimRewardTVS(PROJECT_ID);

        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 0);

        uint256 aliceStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, alice);
        assertEq(aliceStaleIndex, 0);

        assertEq(vesting.getKolTVSRewards(PROJECT_ID, alice), 0);
    }

    function test_H03_ArrayIndexManipulationBug_StaleIndexCausesOutOfBoundsRevert() public {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );

        address[] memory kols = new address[](3);
        kols[0] = alice;
        kols[1] = bob;
        kols[2] = charlie;

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = ALLOCATION_AMOUNT;
        amounts[1] = ALLOCATION_AMOUNT;
        amounts[2] = ALLOCATION_AMOUNT;

        uint256 totalAllocation = ALLOCATION_AMOUNT * 3;
        vesting.setTVSAllocation(PROJECT_ID, totalAllocation, VESTING_PERIOD, kols, amounts);

        vm.prank(alice);
        vesting.claimRewardTVS(PROJECT_ID);

        uint256 aliceStaleIndex = vesting.getKolTVSIndexOf(PROJECT_ID, alice);
        assertEq(aliceStaleIndex, 2);
        assertEq(vesting.getKolTVSAddressesLength(PROJECT_ID), 2);

        vm.expectRevert();
        vesting.getKolTVSAddressAt(PROJECT_ID, aliceStaleIndex);
    }
}

```

### Mitigation


Fix the swap-and-pop logic to properly delete the index mapping and only perform the swap if the element is not already at the last position:

```solidity
function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    uint256 amount = rewardProject.kolTVSRewards[kol];
    require(amount > 0, Caller_Has_No_TVS_Allocation());
    rewardProject.kolTVSRewards[kol] = 0;

    // ... NFT minting and allocation setup ...

    uint256 index = rewardProject.kolTVSIndexOf[kol];
    uint256 arrayLength = rewardProject.kolTVSAddresses.length;

    // Only perform swap if not already the last element
    if (index != arrayLength - 1) {
        address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
        rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
        rewardProject.kolTVSAddresses[index] = lastIndexAddress;
    }

    rewardProject.kolTVSAddresses.pop();
    delete rewardProject.kolTVSIndexOf[kol];  // Clean up the mapping

    emit RewardTVSClaimed(rewardProjectId, kol, nftId, amount, vestingPeriod);
}
```

Apply the same fix to `_claimStablecoinAllocation()`.

  