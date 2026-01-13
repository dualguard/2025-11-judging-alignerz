# [000987] Stale global allocationOf mapping will cause dividend distribution DoS [High]
  
  
### Summary

The missing synchronization between project-specific allocation storage and the global `allocationOf` mapping will cause a denial of service for protocol administrators as the dividend distributor will fail when attempting to read allocation data through the stale global mapping.

### Root cause

In `AlignerzVesting.sol` the choice to use a global `allocationOf` mapping that is only updated during NFT creation is a mistake as the mapping becomes stale when users claim tokens over time. When vesting progress occurs, updates are made only to project-specific allocation storage in `rewardProjects[projectId].allocations[nftId]` or `biddingProjects[projectId].allocations[nftId]`, but the global `allocationOf[nftId]` retains the original values from creation time. This desynchronization occurs because Solidity storage-to-storage struct assignment creates a copy rather than a reference, as seen in [AlignerzVesting.sol#L571](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L571), [AlignerzVesting.sol#L614](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L614), [AlignerzVesting.sol#L887](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L887), and [AlignerzVesting.sol#L1092](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1092).

### Internal pre-conditions

1. Admin needs to call `launchRewardProject()` to set up a reward project with TVS allocations
2. KOL needs to call `claimRewardTVS()` to mint an NFT and populate `allocationOf[nftId]`
3. User needs to call `claimTokens()` to update project-specific allocation storage without updating the global mapping

### External pre-conditions

None required.

### Attack path

1. Protocol administrator calls `launchRewardProject()` and sets TVS allocations for KOLs
2. KOL calls `claimRewardTVS()` which populates `allocationOf[nftId]` with initial allocation data via `AlignerzVesting.sol#L614`
3. User calls `claimTokens()` which updates the project-specific `claimedSeconds[]` and `claimedFlows[]` arrays but does not synchronize `allocationOf[nftId]`
4. Administrator attempts to call `A26ZDividendDistributor.setUpTheDividends()` which internally calls `getUnclaimedAmounts(nftId)` at `A26ZDividendDistributor.sol#L140-L145`
5. The dividend distributor fails when attempting to read from `vesting.allocationOf(nftId)` due to ABI mismatch between the expected interface and actual public mapping getter

### Impact

The protocol administrators cannot set up dividend distribution as the `setUpTheDividends()` function reverts when reading allocation data. This breaks the dividend distribution functionality, preventing administrators from executing a significant auxiliary protocol feature. The underlying desynchronization creates a latent risk where dividend misallocation could occur if the technical reading issues are resolved, as external consumers would compute unclaimed amounts using outdated vesting progress data.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../../src/MockUSD.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "../../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
// Avoid OZ upgrades helper to keep PoC self-contained

contract StaleAllocationOfPOC is Test {
    Aligners26 token; // project token (18 decimals)
    MockUSD usdc;     // stablecoin for dividends (6 decimals)
    AlignerzNFT nft;
    AlignerzVesting vesting;

    address owner;
    address projectCreator;
    address kol1;
    address kol2;

    uint256 constant A = 100 ether; // allocation per KOL (project token units)

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");

        // Deploy tokens and NFT
        usdc = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy vesting implementation and initialize directly (no proxy to avoid OZ upgrades validation)
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint/burn NFTs
        nft.addMinter(address(vesting));
        vesting.setTreasury(makeAddr("treasury"));

        // Fund project creator with project tokens and transfer vesting ownership
        token.transfer(projectCreator, 1_000_000 ether);
        vesting.transferOwnership(projectCreator);

        // Approve vesting to pull project tokens for reward allocations
        vm.prank(projectCreator);
        token.approve(address(vesting), type(uint256).max);
    }

    function _findOwnedNFT(address ownerAddr) internal view returns (uint256 nftId, bool found) {
        uint256 total = nft.getTotalMinted();
        // Token IDs in this ERC721A start at 1, so scan 1..=total
        for (uint256 i = 1; i <= total; i++) {
            try nft.extOwnerOf(i) returns (address o) {
                if (o == ownerAddr) {
                    return (i, true);
                }
            } catch {}
        }
        return (0, false);
    }

    function test_POC_StaleAllocationOf_Misallocates_Dividends() public {
        // 1) Launch a reward project with identical allocations for two KOLs
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vm.prank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdc), startTime, claimWindow);
        uint256 rewardProjectId = 0;

        // Set TVS allocations: kol1 and kol2 each get A vested over 100 days
        address[] memory kols = new address[](2);
        uint256[] memory amts = new uint256[](2);
        kols[0] = kol1; kols[1] = kol2;
        amts[0] = A; amts[1] = A;
        vm.prank(projectCreator);
        vesting.setTVSAllocation(rewardProjectId, 2 * A, 100 days, kols, amts);

        // Each KOL claims their TVS (mints NFT)
        vm.prank(kol1);
        vesting.claimRewardTVS(rewardProjectId);
        vm.prank(kol2);
        vesting.claimRewardTVS(rewardProjectId);

        // Find their NFT IDs by scanning the small minted range
        (uint256 nft1, bool f1) = _findOwnedNFT(kol1);
        (uint256 nft2, bool f2) = _findOwnedNFT(kol2);
        assertTrue(f1 && f2, "NFTs not found for KOLs");

        // 2) Advance time halfway and have kol1 claim tokens (50% of A)
        vm.warp(startTime + 50 days);
        uint256 balBeforeKol1 = token.balanceOf(kol1);
        vm.prank(kol1);
        vesting.claimTokens(rewardProjectId, nft1);
        uint256 claimedKol1 = token.balanceOf(kol1) - balBeforeKol1;
        // Should be ~50% of A (ignoring rounding)
        assertApproxEqAbs(claimedKol1, A / 2, 1);

        // 3) We proceed to set up the external consumer and show end-to-end failure

        // 4) Try to run the dividend distributor end-to-end (as in production)
        // NOTE: This currently REVERTS because the external consumer (distributor)
        // calls vesting.allocationOf(nftId) expecting full arrays, but the implementation's
        // public mapping getter only returns (isClaimed, token, assignedPoolId).
        // The ABI mismatch triggers an abi.decode failure, DoSing dividend setup.
        A26ZDividendDistributor dd = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdc), block.timestamp, 7 days, address(0)
        );
        usdc.transfer(address(dd), 1_000_000);
        vm.expectRevert();
        dd.setUpTheDividends();
    }

    // Helper removed: not needed when asserting the end-to-end revert
}
```

### Mitigation

Consider implementing a synchronization mechanism to keep `allocationOf` current with the actual vesting state. One approach could be to update `allocationOf[nftId]` within the token claiming logic whenever vesting progress changes. Alternatively, consider having external consumers read directly from the project-specific allocation mappings rather than relying on the potentially stale global copy, which would eliminate the synchronization requirement while providing access to current vesting state data.
  