# [000985] Index drift in KOL reward arrays enables wrong-element removal and skipped distributions [Medium]
  
  
### Summary

The incorrect index assignment in `setTVSAllocation()` and `setStablecoinAllocation()` will cause skipped distributions for affected KOLs as multiple allocation calls create mapping drift, leading to wrong-element removal during claims without identity verification.

### Root cause

- In [AlignerzVesting.sol:469](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L469) the index assignment `rewardProject.kolTVSIndexOf[kol] = i` uses the loop counter instead of the current array length plus the counter
- In [AlignerzVesting.sol:494](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L494) the same incorrect pattern exists for stablecoin allocations with `rewardProject.kolStablecoinIndexOf[kol] = i`
- In [AlignerzVesting.sol:615-621](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L615-L621) the swap-and-pop removal mechanism uses stored indices without verifying that the array element at that position matches the claiming KOL's address

### Internal pre-conditions

1. Owner needs to call `setTVSAllocation()` or `setStablecoinAllocation()` multiple times to add KOLs to existing reward projects, causing newly added KOLs to have incorrect index mappings that point to positions occupied by previously added KOLs
2. KOLs with incorrect indices need to attempt claiming their allocations during the claim window

### External pre-conditions

None required.

### Attack path

1. Owner calls `setTVSAllocation()` to add initial KOLs [A, B] to reward project, setting their indices correctly as 0, 1
2. Owner calls `setTVSAllocation()` again to add additional KOLs [C, D] to the same project, but incorrectly assigns indices 0, 1 instead of 2, 3
3. KOL C claims their allocation using `claimRewardTVS()`
4. The contract uses C's stored index (0) without verification, performing swap-and-pop that removes KOL A from the pending array while C remains
5. After the deadline passes, `distributeRemainingRewardTVS()` iterates over the corrupted array, minting a zero-amount NFT to C and valid NFTs to remaining KOLs B and D, while completely skipping A

### Impact

The affected KOLs suffer permanent loss of their allocated rewards as they are wrongly removed from pending arrays and skipped during post-deadline distributions. KOLs who successfully claimed but remain in arrays due to incorrect removal receive meaningless zero-amount NFTs, while wrongly removed KOLs cannot receive their allocations once the claim deadline passes and post-deadline distribution functions rely on the corrupted array state.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {IAlignerzVesting} from "../../src/interfaces/IAlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract RewardIndexDriftPoC is Test {
    Aligners26 internal token;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal ownerAddr;
    address internal KOL_A;
    address internal KOL_B;
    address internal KOL_C;
    address internal KOL_D;

    uint256 internal constant TVS_UNIT = 100 ether; // 100 tokens per KOL

    function setUp() public {
        ownerAddr = address(this);
        KOL_A = makeAddr("KOL_A");
        KOL_B = makeAddr("KOL_B");
        KOL_C = makeAddr("KOL_C");
        KOL_D = makeAddr("KOL_D");

        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        // Allow vesting to mint NFTs
        nft.addMinter(proxy);
        vesting.setTreasury(makeAddr("treasury"));
    }

    function test_IndexDrift_CausesWrongRemoval_And_StuckAllocation() public {
        // 1) Launch reward project (token is vested; stablecoin unused here)
        uint256 startTime = block.timestamp + 1; // arbitrary
        uint256 claimWindow = 1 hours;
        vesting.launchRewardProject(address(token), address(0xBEEF), startTime, claimWindow);

        // 2) Owner approves token to vesting for total allocations (4 * 100 = 400)
        token.approve(address(vesting), 400 ether);

        // 3) First allocation call adds [A, B]
        address[] memory addrs1 = new address[](2);
        addrs1[0] = KOL_A;
        addrs1[1] = KOL_B;
        uint256[] memory amts1 = new uint256[](2);
        amts1[0] = TVS_UNIT;
        amts1[1] = TVS_UNIT;
        vesting.setTVSAllocation(0, 2 * TVS_UNIT, 30 days, addrs1, amts1);

        // 4) Second allocation call adds [C, D]
        //    BUG: kolTVSIndexOf[...] is set to loop index i (0,1), not the actual array positions (2,3)
        address[] memory addrs2 = new address[](2);
        addrs2[0] = KOL_C;
        addrs2[1] = KOL_D;
        uint256[] memory amts2 = new uint256[](2);
        amts2[0] = TVS_UNIT;
        amts2[1] = TVS_UNIT;
        vesting.setTVSAllocation(0, 2 * TVS_UNIT, 30 days, addrs2, amts2);

        // 5) C claims during window. Due to stale index mapping, swap-and-pop removes WRONG element (A),
        //    while C remains in the array and gets kolTVSRewards[C] = 0 after minting.
        vm.prank(KOL_C);
        vesting.claimRewardTVS(0);

        // 6) Move past deadline and distribute remaining via array-driven function.
        vm.warp(startTime + claimWindow + 1);
        vesting.distributeRemainingRewardTVS(0);

        // 7) Inspect minted NFTs and allocations via events and ownership
        uint256 totalMinted = nft.getTotalMinted();
        assertEq(totalMinted, 4, "Expected 4 NFTs: C(claim) + C(0-amount) + B + D");

        // Ownership checks: C has 2 NFTs; B and D have 1; A has 0
        assertEq(nft.balanceOf(KOL_C), 2, "Claimer C ended with two NFTs (one due to wrong removal)");
        assertEq(nft.balanceOf(KOL_B), 1, "KOL B received one NFT via distribution");
        assertEq(nft.balanceOf(KOL_D), 1, "KOL D received one NFT via distribution");
        assertEq(nft.balanceOf(KOL_A), 0, "KOL A incorrectly received nothing due to wrong removal");

        // Re-run distribution in a fresh log recording to decode event amounts and prove 0-amount mint to C occurred
        // Note: No-op if arrays already empty; we already saw from trace this emits C with amount=0 then B and D with 100 each
        // To capture the first distribution we must record before calling; so we simulate again from scratch:
        // For succinctness, we assert the effect using the ownership changes above and deadline denial below.

        // 8) Demonstrate KOL A is now blocked from claiming (deadline passed)
        vm.startPrank(KOL_A);
        vm.expectRevert();
        vesting.claimRewardTVS(0); // reverts with Deadline_Has_Passed
        vm.stopPrank();
    }
}
```

### Mitigation

Consider implementing identity verification in the claim functions before performing swap-and-pop operations and correcting the index assignment logic in allocation functions. One approach could be to validate that the array element at the stored index matches the claiming KOL's address, reverting if there's a mismatch. For the identity verification, the claim functions could include a check such as:

```diff
  uint256 index = rewardProject.kolTVSIndexOf[kol];
+ require(rewardProject.kolTVSAddresses[index] == kol, "Index mismatch");
  uint256 arrayLength = rewardProject.kolTVSAddresses.length;
```

For the index assignment correction, the allocation functions should track the existing array length:

```diff
+ uint256 startIndex = rewardProject.kolTVSAddresses.length;
  for (uint256 i = 0; i < length; i++) {
      address kol = kolTVS[i];
      rewardProject.kolTVSAddresses.push(kol);
      uint256 amount = TVSamounts[i];
      rewardProject.kolTVSRewards[kol] = amount;
-     rewardProject.kolTVSIndexOf[kol] = i;
+     rewardProject.kolTVSIndexOf[kol] = startIndex + i;
      totalAmount += amount;
```
  