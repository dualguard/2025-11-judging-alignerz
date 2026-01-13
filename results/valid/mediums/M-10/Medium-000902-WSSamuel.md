# [000902] Excessive TVS Merges Cause Permanent Claim DoS
  
  ### Summary

The `mergeTVS()` function allows users to merge **an unlimited number** of TVS NFTs into a single NFT. Each merge appends new entries to multiple flow-related arrays, reaching a limit where `claimTokens()` will fail with a `MemoryLimitOOG` error. This permanently prevents users from claiming tokens for that TVS NFT.

### Root Cause

In [`mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) appends storage to the `mergedNFTId`:

```solidity
mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
```

Because these are dynamic arrays in storage, invoking `claimTokens()` requires loading the entire array into memory. In my test, at around **350 entries**, memory expansion reaches EVM limits and the function reverts with `MemoryLimitOOG`. This causes a **permanent DoS** for the merged NFT, as no mechanism exists to remove or reduce flows.

### Internal Pre-conditions

1. User needs to merge over 350 TVS's together

### External Pre-conditions

N/A

### Attack Path

1. User buys multiple TVS' and to store them, he merges them as it costs no fee to do so.
2. User then attempts to `claimTokens`, but it will revert, making the TVS completely useless.

### Impact

TVS becomes **irredeemably bricked** if user merges too many tokens. Importantly, `splitTVS` doesn't reverse this issue.

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_too_many_merged_tvs_dos -vvvv
```

First, 2 different bugs needs to be fixed, so in `FeesManager::calculateFeeAndNewAmountForOneTVS` fix the following code:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+ newAmounts = new uint256[](length);
+ for (uint256 i; i < length; i++) {
- for (uint256 i; i < length;) {
```

You will see in the test that when claiming a TVS with over 350 tokens merged in it, it will revert due to `[MemoryLimitOOG] EvmError: MemoryLimitOOG`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {A26ZDividendDistributor} from "src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public distributor;
    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();

        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdt), block.timestamp, 1 weeks, address(token)
        );
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        usdt.mint(projectCreator, 2e18);

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_too_many_merged_tvs_dos() public {
        uint256 numKOLs = 350;
        address[] memory KOL = new address[](numKOLs);

        for (uint256 i = 0; i < numKOLs; i++) {
            KOL[i] = makeAddr(string(abi.encodePacked("KOL_", vm.toString(i))));
        }
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(100);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 3.5e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolTVS = new address[](numKOLs);
        for (uint256 i = 0; i < KOL.length; i++) {
            kolTVS[i] = KOL[i];
        }
        uint256[] memory TVSamounts = new uint256[](numKOLs);
        for (uint256 i = 0; i < KOL.length; i++) {
            TVSamounts[i] = 3.5e18 / numKOLs;
        }

        vesting.setTVSAllocation(0, 3.5e18, 5 weeks, kolTVS, TVSamounts);
        vm.stopPrank();
        vm.prank(KOL[0]);
        vesting.claimRewardTVS(0);
        for (uint256 tokenId = 1; tokenId <= KOL.length - 1; tokenId++) {
            vm.prank(KOL[tokenId]);
            vesting.claimRewardTVS(0);
            vm.prank(KOL[tokenId]);
            nft.transferFrom(KOL[tokenId], KOL[0], tokenId + 1);
        }
        vm.startPrank(KOL[0]);
        uint256[] memory projectIds = new uint256[](numKOLs - 1);
        for (uint256 i = 0; i < KOL.length - 1; i++) {
            projectIds[i] = 0;
        }

        uint256[] memory nftIds = new uint256[](numKOLs - 1);
        for (uint256 i = 0; i < KOL.length - 1; i++) {
            nftIds[i] = i + 2;
        }

        vesting.mergeTVS(0, 1, projectIds, nftIds);
        vm.warp(5 weeks);
        uint256 balanceBefore = token.balanceOf(address(KOL[0]));
        vesting.claimTokens(0, 1);
        uint256 balanceAfter = token.balanceOf(address(KOL[0]));
        console.log("Claimed :%e", balanceAfter - balanceBefore);
    }
}
```

### Mitigation

Add a hard cap on maximum flows

```solidity
uint256 constant MAX_FLOWS = 100;
require(
    mergedTVS.amounts.length + newFlows <= MAX_FLOWS,
    TooManyFlows()
);
```
  