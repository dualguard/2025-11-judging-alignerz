# [000903] mergeTVS Allows Self-Merge resulting in loss of user funds
  
  ### Summary

`mergeTVS()` does not validate that the target NFT (`mergedNftId`) is _excluded_ from the list of source `nftIds`. If a mistaken caller includes `mergedNftId` in `nftIds`, the user will lose their NFT, as their `mergedNftId` would be burned.

### Root Cause

Inside [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) the code iterates over `nftIds` and calls:

```solidity
feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
```

There is **no check** to ensure:

```
nftIds[i] != mergedNftId
```

### Internal Pre-conditions

- Caller is the owner of `mergedNftId` (required earlier in function).
- Caller passes `nftIds` where one element equals `mergedNftId` (accidentally or maliciously).

### External Pre-conditions

N/A

### Attack Path

1. Owner calls merge NFT, mistakenly putting their `mergedNftId` into the array of `nftIds`
2. `_merge` will burn `mergedNftId`, leaving the owner with nothing.

### Impact

An unsuspecting owner can permanently lose their NFT and allocations.

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_merge_with_self -vvvv
```

First, 2 different bugs needs to be fixed, so in `FeesManager::calculateFeeAndNewAmountForOneTVS` fix the following code:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+ newAmounts = new uint256[](length);
+ for (uint256 i; i < length; i++) {
- for (uint256 i; i < length;) {
```

You will see in the test that `mergeTVS` doesn't revert if `nftIds` contains the `mergedNftId`, in addition, to show that the TVS has burned, we will attempt to claim the token, but it will revert as it was burned.

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

    function test_merge_with_self() public {
        address KOL1 = makeAddr("KOL1");

        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(100);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 1e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolTVS = new address[](2);
        kolTVS[0] = KOL1;
        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 1e18;

        vesting.setTVSAllocation(0, 1e18, 5 weeks, kolTVS, TVSamounts);
        vm.stopPrank();
        vm.startPrank(KOL1);
        vesting.claimRewardTVS(0);

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 0;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = 1;
        vesting.mergeTVS(0, 1, projectIds, nftIds);
        vm.warp(5 weeks);
        //e Time passes, we attempt to claim it, but since the token was burned in the merge, we can't
        //e complete fund loss of user
        vm.expectRevert();
        vesting.claimTokens(0, 1);
    }
}
```

### Mitigation

Add a check to ensure 

```
nftIds[i] != mergedNftId
```
  