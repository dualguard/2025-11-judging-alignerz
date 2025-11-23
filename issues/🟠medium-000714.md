# [000714] Snapshot-based dividends ignore claimed flows and miscredit users
  
  ## Summary
A26ZDividendDistributor computes dividends using allocationOf(nftId), a snapshot set at NFT creation/split and never synchronized after vesting claims or merges.
 As vesting progresses, project-level allocations are updated while allocationOf remains stale, causing getUnclaimedAmounts to overestimate “unclaimed” amounts and over-allocate dividends to users who already claimed.

## Finding Description
- The vesting contract maintains authoritative state within project-level mappings for each NFT: biddingProjects[projectId].allocations[nftId] and rewardProjects[projectId].allocations[nftId]. The struct includes amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, isClaimed, token, and assignedPoolId (`protocol/src/contracts/vesting/AlignerzVesting.sol:69–80`).
- A separate mapping allocationOf[nftId] exposes an Allocation snapshot and is only written:
  - On bidding NFT claim (`protocol/src/contracts/vesting/AlignerzVesting.sol:878–888`).
  - On reward TVS claim (`protocol/src/contracts/vesting/AlignerzVesting.sol:607–615`).
  - On distributing remaining reward TVS (`protocol/src/contracts/vesting/AlignerzVesting.sol:563–572`).
  - On split (`protocol/src/contracts/vesting/AlignerzVesting.sol:1086–1093`).
- When vesting progress occurs:
  - claimTokens updates the authoritative Allocation (increments claimedSeconds, toggles claimedFlows, sets isClaimed, and burns the NFT if complete) but never updates allocationOf[nftId] [(`protocol/src/contracts/vesting/AlignerzVesting.sol:954–975`)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975).
  - Merges copy flows from other NFTs into the survivor and burn the merged NFTs, again without updating any allocationOf snapshot for the survivor (`protocol/src/contracts/vesting/AlignerzVesting.sol:1037–1048`).
- The dividend distributor uses `allocationOf` to compute “unclaimed” amounts and allocate `stablecoin` to owners:
  - `getUnclaimedAmounts()` loads fields from `allocationOf(nftId)` and treats `claimedSeconds == 0` as fully unclaimed; if partial claims occurred but the snapshot wasn’t updated, it inflates unclaimed amounts [(`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:140–161`)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161).
  - Aggregates across minted NFTs in `getTotalUnclaimedAmounts()` and allocates owner balances in `_setDividends()` using `unclaimedAmountsIn` (`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126–136`, `214–218`).
- As stated: “allocationOf is a snapshot taken when the NFT is created or split and is never updated when vesting claims or merges happen.” That is consistent with the code paths above and explains why the distributor reads obsolete vesting progress.
- The user’s PoC notes that to execute end-to-end they “Fixed the ABI mismatch of `allocationOf` so it returns the full `Allocation` struct” and “Fixed the infinite loop in `getUnclaimedAmounts()` (missing `i++` on continue paths).” Those are separate issues; once the loop and ABI are corrected, the stale-snapshot misallocation remains and is exploitable through normal operation.
- Affected function code
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1028-L1047

```solidity
// protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
// Always reads from the stale snapshot:
uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
// If claimedSeconds[i] is 0 (stale), treats full amount as unclaimed
if (claimedSeconds[i] == 0) {
    amount += amounts[i];  // Overpayment!
}    
}

// protocol/src/contracts/vesting/AlignerzVesting.sol
function claimTokens(uint256 projectId, uint256 nftId) external {
    ...
// In claimTokens():
(Allocation storage allocation, IERC20 token) = isBiddingProject ? 
    (biddingProjects[projectId].allocations[nftId], ...) : 
    (rewardProjects[projectId].allocations[nftId], ...);

// Updates claimedSeconds/claimedFlows in `allocation` (project-level)
allocation.claimedSeconds[i] += claimableSeconds;
if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
    allocation.claimedFlows[i] = true;
}
// No update to allocationOf[nftId]!    
}

// protocol/src/contracts/vesting/AlignerzVesting.sol
function _merge(Allocation storage mergedTVS, uint256 projectId, uint256 nftId, IERC20 token) internal returns (uint256 feeAmount) {
    ...
    // In _merge():
mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
// No allocationOf[mergedNftId] update!
}
```

- Links:
  - Allocation struct and mappings: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L71-L80
  - Distributor logic: `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:140–161`, `214–218`
  - Snapshot writes: `protocol/src/contracts/vesting/AlignerzVesting.sol:563–572`, `607–615`, `878–888`, `1086–1093`

**Attack Path**
- Preconditions:
  - A bidder or KOL has a TVS NFT with flows recorded under allocations.
  - The user makes partial or full claims via claimTokens, updating claimedSeconds/claimedFlows in authoritative storage only.
- Steps:
  - The owner later funds A26ZDividendDistributor and runs setAmounts/setDividends.
  - getUnclaimedAmounts reads allocationOf(nftId) snapshot where claimedSeconds[i] still equals 0 and claimedFlows[i] is false.
  - The distributor treats entire 
  - amounts[i] as unclaimed or underestimates claimed portions, inflating unclaimedAmountsIn[nftId].
  - _setDividends allocates stablecoin proportionally, overpaying holders who previously claimed.
- Highest-impact scenario:
  - A TVS NFT with multiple flows has fully claimed one or more flows, and partially claimed others. Snapshot remains at initial state. Distributor allocates dividends based on full `amounts[i]` across those flows, significantly over-crediting that owner relative to peers whose flows are actually unclaimed.

## Impact
- Medium — funds are not stolen directly from contracts, but dividend distribution becomes economically incorrect, overpaying some holders and underpaying others under normal operation once dividends are configured.
## poc 

How to Run

- Commands used:
  - forge clean
  - forge build
  - forge test --mt test_SnapshotStale_MiscreditRisk -vvv

import
```diff
- import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
+ import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
+ import {Upgrades, UnsafeUpgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
```
add here 
`Line  45    function setUp() public `
import
```diff
- address payable proxy = payable(Upgrades.deployUUPSProxy(
-        "AlignerzVesting.sol",


 + address payable proxy = payable(UnsafeUpgrades.deployUUPSProxy(
 +          address(new AlignerzVesting()),
```
```solidity
function test_SnapshotStale_MiscreditRisk() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
        vm.stopPrank();

        address bidder = bidders[0];
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(bidder, BIDDER_USD, PROJECT_ID, 0);
        leaves[1] = getLeaf(makeAddr("dummy"), BIDDER_USD, PROJECT_ID, 0);
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof = m.getProof(leaves, 0);
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof);

        vm.warp(block.timestamp + 45 days);
        uint256 balBefore = token.balanceOf(bidder);
        vm.recordLogs();
        vm.prank(bidder);
        vesting.claimTokens(PROJECT_ID, nftId);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 sig = keccak256(bytes("TokensClaimed(uint256,bool,uint256,bool,uint256,uint256[],uint256,address,uint256[])"));
        uint256 claimedSec = 0;
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].topics[0] == sig) {
                bool _isB;
                bool _isC;
                uint256[] memory cs;
                uint256 _ts;
                address _user;
                uint256[] memory _amts;
                (_isB, _isC, cs, _ts, _user, _amts) = abi.decode(logs[i].data, (bool,bool,uint256[],uint256,address,uint256[]));
                claimedSec = cs[0];
                break;
            }
        }
        uint256 balAfter = token.balanceOf(bidder);
        assertTrue(balAfter > balBefore);
        assertTrue(claimedSec > 0);
        uint256 wrongUnclaimed = BIDDER_USD;
        uint256 correctUnclaimed = BIDDER_USD - (BIDDER_USD * claimedSec / (90 days));
        assertGt(wrongUnclaimed, correctUnclaimed);
    }
```

## Recommendation
- Keep allocationOf[nftId] synchronized with authoritative storage whenever vesting state mutates (claims and merges). Sync after claimTokens updates and after mergeTVS finalizes. Optionally clear snapshots for burned NFTs.

  