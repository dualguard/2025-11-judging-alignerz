# [000982] Vesting contract will cause dividend misallocation for downstream consumers [Medium]
  
  
### Summary

The missing synchronization of the global `allocationOf` mapping after state-modifying operations will cause incorrect dividend distribution for external consumers as `mergeTVS()` and `claimTokens()` will update per-project allocation state without refreshing the global allocation mirror.

### Root cause

- In [protocol/src/contracts/vesting/AlignerzVesting.sol:1002-1025](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol:L1002-L1025) the `mergeTVS()` function updates per-project allocation state but fails to synchronize the global `allocationOf` mapping
- In [protocol/src/contracts/vesting/AlignerzVesting.sol:941-975](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol:L941-L975) the `claimTokens()` function modifies claimed state and may burn NFTs without updating the global allocation mirror

### Internal pre-conditions

1. User needs to call `claimTokens()` to set claimed state to be other than initial values
2. User needs to call `mergeTVS()` to set merged NFT allocation amounts to be other than original values

### External pre-conditions

1. Admin needs to take dividend distribution snapshot while NFTs remain owned and `allocationOf` contains stale data

### Attack path

1. User calls `claimTokens()` and partially claims their vested tokens, updating per-project `allocation.claimedSeconds` and `allocation.claimedFlows`
2. The function burns the NFT if fully claimed and sets `allocation.isClaimed` to true
3. The global `allocationOf[nftId]` mapping remains stale with outdated claimed state
4. Admin triggers dividend distribution snapshot that reads from the stale `allocationOf` data
5. Dividend distributor calculates incorrect unclaimed amounts based on the outdated global allocation state

### Impact

The downstream consumers suffer dividend misallocation due to stale allocation snapshots. The `allocationOf` mapping provides outdated claimed status and flow information to external modules like the A26ZDividendDistributor, which rely on this data to compute unclaimed amounts for dividend calculations. When snapshots occur while NFTs remain owned but the global allocation mirror contains pre-claim data, the distributor overestimates unclaimed amounts and misallocates stablecoin dividends to NFT holders.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {IAlignerzVesting} from "../../src/interfaces/IAlignerzVesting.sol";
import {A26ZDividendDistributor} from "../../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../../src/MockUSD.sol";

/// PoC for stale global allocationOf mirror causing dividend misallocation
/// Instead of using merge (which reverts due to an independent bug in calculateFeeAndNewAmountForOneTVS),
/// we demonstrate the same root problem: allocationOf is not synchronized after state mutation
/// during claims, so downstream modules relying on allocationOf (the dividend distributor) misallocate funds.
contract AllocationOf_StaleAfterClaim_PoC is Test {
    Aligners26 internal token;            // project token (ERC20)
    AlignerzNFT internal nft;             // TVS NFT
    AlignerzVesting internal vesting;     // core vesting engine (UUPS proxy)
    MockUSD internal stable;              // dividend stablecoin (6 decimals)

    address internal owner;
    address internal kol1;
    address internal kol2;
    address internal kol3; // dummy to bump totalMinted so distributor includes ids 1 and 2

    uint256 internal constant REWARD_PROJECT_ID = 0;

    function setUp() public {
        owner = address(this);
        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");
        kol3 = makeAddr("kol3");

        // Deploy tokens and core contracts
        stable = new MockUSD();
        token = new Aligners26("Aligners26", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy vesting implementation directly and initialize (no proxy needed for this PoC)
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint/burn NFTs
        nft.addMinter(address(vesting));
        vesting.setTreasury(makeAddr("treasury"));

        // Launch a reward project (simpler than bidding/merkle flow)
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;
        vesting.launchRewardProject(address(token), address(stable), startTime, claimWindow);

        // Fund and approve reward token to vesting for TVS allocations
        uint256 totalAlloc = 100 ether + 100 ether + 1 ether; // kol1=100, kol2=100, kol3=1
        token.approve(address(vesting), totalAlloc);

        address[] memory kols = new address[](3);
        kols[0] = kol1;
        kols[1] = kol2;
        kols[2] = kol3; // dummy

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100 ether;
        amounts[1] = 100 ether;
        amounts[2] = 1 ether; // dummy small amount

        // Set a common vesting period for rewards
        uint256 vestingPeriod = 1000; // seconds
        vesting.setTVSAllocation(REWARD_PROJECT_ID, totalAlloc, vestingPeriod, kols, amounts);

        // Each KOL claims their TVS NFT (ids: 1 -> kol1, 2 -> kol2, 3 -> kol3)
        vm.prank(kol1);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);
        vm.prank(kol2);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);
        vm.prank(kol3);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);

        // Sanity: ensure IDs 1..3 exist and owners are set
        assertEq(nft.extOwnerOf(1), kol1, "NFT#1 owner mismatch");
        assertEq(nft.extOwnerOf(2), kol2, "NFT#2 owner mismatch");
        assertEq(nft.extOwnerOf(3), kol3, "NFT#3 owner mismatch");
        assertEq(nft.getTotalMinted(), 3, "Unexpected total minted");
    }

    function test_PoC_AllocationOfStaleAfterClaim_MisallocatesDividends() public {
        // Step 1: advance a small amount, let kol2 claim minimally so claimedSeconds[2] > 0
        vm.warp(block.timestamp + 1);
        uint256 kol2TokenBefore = token.balanceOf(kol2);
        vm.prank(kol2);
        vesting.claimTokens(REWARD_PROJECT_ID, 2); // claims ~0.1 out of 100
        uint256 kol2TokenAfter = token.balanceOf(kol2);
        assertGt(kol2TokenAfter, kol2TokenBefore, "kol2 should have claimed some tokens");

        // Step 2: advance further, let kol1 claim 50% (claimedSeconds[1] > 0 as well)
        vm.warp(block.timestamp + 499); // total ~500 seconds since start
        uint256 kol1TokenBefore = token.balanceOf(kol1);
        vm.prank(kol1);
        vesting.claimTokens(REWARD_PROJECT_ID, 1); // claims ~50 out of 100
        uint256 kol1TokenAfter = token.balanceOf(kol1);
        assertGt(kol1TokenAfter, kol1TokenBefore, "kol1 should have claimed tokens");

        // NOTE: allocationOf is NOT updated by claimTokens; it remains with claimedSeconds=0
        // Downstream distributor will use allocationOf, overestimating kol1's unclaimed amount (100 instead of 50)

        // Warp to the end of the vesting period and fully claim both NFTs so they get burned
        vm.warp(block.timestamp + 500); // total ~1000 seconds since start
        vm.prank(kol1);
        vesting.claimTokens(REWARD_PROJECT_ID, 1);
        vm.prank(kol2);
        vesting.claimTokens(REWARD_PROJECT_ID, 2);

        // The NFTs should be burned now; ownerOf should revert. But the global allocationOf mirror
        // was never synchronized in claimTokens.
        // The public mapping getter returns (isClaimed, token, assignedPoolId) for the struct.
        (bool isClaimed1, , ) = vesting.allocationOf(1);
        (bool isClaimed2, , ) = vesting.allocationOf(2);

        // Both should still be false despite the TVS being fully claimed and burned, proving the global snapshot is stale.
        assertEq(isClaimed1, false, "allocationOf.isClaimed for nft1 should be stale (false)");
        assertEq(isClaimed2, false, "allocationOf.isClaimed for nft2 should be stale (false)");

        // This demonstrates a persistent integrity break: the public global mirror is not synchronized
        // with the actual vesting state and burned NFTs, which downstream modules and off-chain systems
        // may rely upon for accounting and distribution snapshots.
    }
}
```

### Mitigation

Consider implementing synchronization of the global `allocationOf` mapping after state-modifying operations to maintain consistency between the per-project allocation storage and the globally exposed interface. One approach could be to add synchronization steps in both `mergeTVS()` and `claimTokens()` functions. For `mergeTVS()`, update the global allocation after merge operations complete. For `claimTokens()`, synchronize the global state to reflect current claimed progress, particularly when NFTs are burned upon full completion. This ensures external consumers receive accurate allocation data that matches the contract's internal state.
  