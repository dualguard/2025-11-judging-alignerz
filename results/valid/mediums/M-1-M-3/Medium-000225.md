# [000225] Stale `allocationOf` snapshots let partially claimed TVSs overstate dividend weight
  
  ### Summary

`allocationOf[nftId]` is only populated once when the NFT is minted and is never refreshed, so even after claiming part of a vesting flow, `A26ZDividendDistributor` still sees `claimedSeconds = 0` and over-allocates future stablecoin distributions to that holder.

### Root Cause

Whenever an NFT is minted `protocol/src/contracts/vesting/AlignerzVesting.sol:`([571](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L571), [614](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L614), [887](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L887), [1092](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1092)), the contract takes a one-time snapshot of the allocation struct into `allocationOf[nftId]`. Later lifecycle routines like [claimTokens](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975), [mergeTVS/_merge](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1048), and [splitTVS/_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1141) mutate only the live project allocations and never refresh that snapshot. Because storage-to-storage assignment deep-copies the dynamic arrays, `allocationOf` never reflects subsequent updates to `claimedSeconds/claimedFlows`. Yet [A26ZDividendDistributor](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) relies exclusively on `vesting.allocationOf(nftId)` when computing unclaimed balances, so dividend math remains permanently skewed by stale data.

### Internal Pre-conditions

1. An NFT holder claims part of their vesting flow (so live storage increments `claimedSeconds` while the NFT still exists).
2. The NFT is not fully claimed, thus remains in circulation and is considered during dividend distribution.
3. Treasury later calls `setUpTheDividends`, which reads `vesting.allocationOf(nftId)` to determine each holder’s unclaimed amount.

### External Pre-conditions

None

### Attack Path

1. KOL mints a 100-token TVS NFT and immediately claims 50% of the vesting flow (because the vesting start time is in the past).
2. `rewardProjects[0].allocations[nftId].claimedSeconds[0]` becomes 50, so calling `claimTokens` again reverts with `No_Claimable_Tokens`.
3. The shadow snapshot `allocationOf[nftId].claimedSeconds[0]` remains `0` because it was never refreshed after minting.
4. Dividend distributor calls `_setAmounts()` and, seeing `claimedSeconds = 0`, counts the entire 100 tokens as “unclaimed,” inflating this holder’s dividend weight at the expense of others.

### Impact

Dividend pools systematically overpay any partially claimed NFT by the exact amount already redeemed, reducing the share owed to honest holders. A fully claimed-but-not-burned NFT could keep draining future distributions until the snapshot is refreshed or the NFT is burned.

### PoC

1. Create `protocol/test/AllocationSnapshotDrift.t.sol`.
2. Paste the following harnessed test and helper contracts.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";

contract AllocationSnapshotDriftTest is Test {
    AlignerzVestingExposed internal vesting;
    AlignerzNFT internal nft;
    MockUSD internal token;

    uint256 internal constant TVS_AMOUNT = 100 * 1e6; // MockUSD has 6 decimals
    uint256 internal constant VESTING_PERIOD = 100;
    uint256 internal nftId;
    uint256 internal rewardStart;
    address internal kol = address(0xBEEF);

    function setUp() public {
        nft = new AlignerzNFT("Alignerz", "ALGN", "ipfs://alignerz/");
        vesting = new AlignerzVestingExposed();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));

        token = new MockUSD();

        // Launch a reward project whose vesting start time is recorded.
        rewardStart = block.timestamp;
        vesting.launchRewardProject(address(token), address(token), rewardStart, 30 days);

        // Configure a single KOL allocation.
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TVS_AMOUNT;

        token.approve(address(vesting), TVS_AMOUNT);
        vesting.setTVSAllocation(0, TVS_AMOUNT, VESTING_PERIOD, kols, amounts);

        // Mint the reward TVS NFT to the KOL address.
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        nftId = nft.getTotalMinted(); // token IDs start at 1
        assertEq(nft.ownerOf(nftId), kol, "KOL should own the minted NFT");
    }

    function testAllocationSnapshotRemainsZeroAfterPartialClaim() public {
        vm.warp(rewardStart + VESTING_PERIOD / 2);

        vm.startPrank(kol);
        // First claim consumes half the vesting period and should succeed.
        vesting.claimTokens(0, nftId);
        vm.stopPrank();

        // Live allocation reflects the claimed seconds.
        (uint256[] memory liveClaimedSeconds, bool[] memory liveClaimedFlows) =
            vesting.getLiveClaimData(0, nftId);
        assertEq(liveClaimedSeconds[0], VESTING_PERIOD / 2, "live allocation didn't advance claimed seconds");
        assertEq(liveClaimedFlows[0], false, "live allocation incorrectly marked flow as claimed");

        // Snapshot exposed via allocationOf never updated, still reporting zero claimed seconds.
        (uint256[] memory claimedSecondsSnapshot, bool[] memory claimedFlowsSnapshot) =
            vesting.getSnapshotClaimData(nftId);
        assertEq(claimedSecondsSnapshot[0], 0, "allocationOf unexpectedly tracked claimed seconds");
        assertEq(claimedFlowsSnapshot[0], false, "allocationOf unexpectedly flipped claimed flow flag");
    }
}

contract AlignerzVestingExposed is AlignerzVesting {
    function getLiveClaimData(uint256 rewardProjectId, uint256 nftId)
        external
        view
        returns (uint256[] memory, bool[] memory)
    {
        Allocation storage alloc = rewardProjects[rewardProjectId].allocations[nftId];
        uint256 len = alloc.claimedSeconds.length;
        uint256[] memory claimed = new uint256[](len);
        bool[] memory flows = new bool[](len);
        for (uint256 i; i < len;) {
            claimed[i] = alloc.claimedSeconds[i];
            flows[i] = alloc.claimedFlows[i];
            unchecked {
                ++i;
            }
        }
        return (claimed, flows);
    }

    function getSnapshotClaimData(uint256 nftId) external view returns (uint256[] memory, bool[] memory) {
        Allocation storage alloc = allocationOf[nftId];
        uint256 len = alloc.claimedSeconds.length;
        uint256[] memory claimed = new uint256[](len);
        bool[] memory flows = new bool[](len);
        for (uint256 i; i < len;) {
            claimed[i] = alloc.claimedSeconds[i];
            flows[i] = alloc.claimedFlows[i];
            unchecked {
                ++i;
            }
        }
        return (claimed, flows);
    }
}
```
3. Run `forge test --match-test testAllocationSnapshotRemainsZeroAfterPartialClaim`.

### Mitigation

Whenever an allocation changes (`claimTokens`, `_merge`, `_split`, reward distributions), also update `allocationOf[nftId]`, or remove the redundant mapping entirely and expose read-only views into the live `rewardProjects`/`biddingProjects` state to avoid desynchronization.
  