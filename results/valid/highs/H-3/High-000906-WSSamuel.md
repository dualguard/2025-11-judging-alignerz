# [000906] splitTVS Fails Due to Uninitialized Allocation Arrays in _computeSplitArrays
  
  ### Summary

The `splitTVS()` function calls `_computeSplitArrays()` to calculate per-flow allocations for new split NFTs. `_computeSplitArrays()` attempts to write into memory arrays (`alloc.amounts`, `alloc.vestingPeriods`, etc.) **without allocating them**, causing **array out-of-bounds access (panic 0x32)**. As a result, **all calls to `splitTVS()` revert**, preventing NFT owners from splitting their TVS allocations.

### Root Cause

Inside [`_computeSplitArrays()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113):

```solidity
for (uint256 j; j < nbOfFlows;) {
    alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
    alloc.vestingPeriods[j] = baseVestings[j];
    alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
    alloc.claimedSeconds[j] = baseClaimed[j];
    alloc.claimedFlows[j] = baseClaimedFlows[j];
    unchecked { ++j; }
}
```

- `alloc` is a **memory struct** returned from the function.
- Its arrays (`amounts`, `vestingPeriods`, etc.) are **never initialized** with `new uint256[](nbOfFlows)` or `new bool[](nbOfFlows)`.
- Writing to any index `j = 0` immediately causes **panic 0x32**.

Since `splitTVS()` relies on `_computeSplitArrays()`, the entire transaction reverts when the first allocation write occurs.

### Internal Pre-conditions

- `msg.sender` owns a `NFT/TVS` and calls `splitTVS`

### External Pre-conditions

N/A

### Attack Path

1. User who owns a TVS attempts split it via `splitTVS`

### Impact

- NFT owners **cannot split their TVS allocations at all**

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_merge_TVS -vv
```

First, 2 different bugs needs to be fixed, so in `FeesManager::calculateFeeAndNewAmountForOneTVS` fix the following code:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+ newAmounts = new uint256[](length);
+ for (uint256 i; i < length; i++) {
- for (uint256 i; i < length;) {
```

Now when running the test, you will see that `splitTVS` will revert with a `[FAIL: panic: array out-of-bounds access (0x32)]` different from the one caused by `calculateFeeAndNewAmountForOneTVS`. 

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

    function test_merge_TVS() public {
        address KOL1 = makeAddr("KOL1");
        address KOL2 = makeAddr("KOL2");

        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 1e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolTVS = new address[](2);
        kolTVS[0] = KOL1;
        kolTVS[1] = KOL2;
        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 5e17;
        TVSamounts[1] = 5e17;
        vesting.setTVSAllocation(0, 1e18, 5 weeks, kolTVS, TVSamounts);
        vm.stopPrank();
        vm.startPrank(KOL1);
        vesting.claimRewardTVS(0);
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;
        vesting.splitTVS(0, percentages, 1);
    }
}
```

### Mitigation

Allocate memory arrays for all fields of the returned `Allocation` before writing:

```solidity
alloc.amounts = new uint256[](nbOfFlows);
alloc.vestingPeriods = new uint256[](nbOfFlows);
alloc.vestingStartTimes = new uint256[](nbOfFlows);
alloc.claimedSeconds = new uint256[](nbOfFlows);
alloc.claimedFlows = new bool[](nbOfFlows);
```

This ensures `_computeSplitArrays()` can safely write to each index `j` and return a valid memory struct.

  