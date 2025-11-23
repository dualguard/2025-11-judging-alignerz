# [000905] splitTVS Miscalculates Amounts When Splitting Multiple TVS
  
  ### Summary

The `_computeSplitArrays()` function is intended to split a TVS allocation by a given `percentage`. When splitting into **multiple new TVS**, it incorrectly uses the same storage `allocation` pointer for every iteration. This causes **progressive reduction** of the base amounts: each subsequent split operates on already-reduced values, resulting in **losing half (or more) of the intended allocation per split**.

### Root Cause

Inside [`_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113):

```solidity
uint256[] memory baseAmounts = allocation.amounts;

alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```

- `baseAmounts` is read directly from `allocation.amounts` (storage).
- In [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1088), the loop iterates over multiple percentages and overwrites the `allocation.amounts` after the first iteration.
- Subsequent iterations now read `baseAmounts` from the **already-reduced allocation**, not the original amount.

**Example:**

| Step | Original Amount | Percentage | Calculated Split | Notes                             |
| ---- | --------------- | ---------- | ---------------- | --------------------------------- |
| 1    | 5e17            | 50%        | 2.5e17           | Correct first split               |
| 2    | 2.5e17          | 50%        | 1.25e17          | Incorrect: should still be 2.5e17 |

This effect compounds with more splits, causing a **progressive loss of allocation**.


### Internal Pre-conditions

- `splitTVS()` is called with `percentages.length > 1`
- Several bugs needs to be fixed in `calculateFeeAndNewAmountForOneTVS` and `_computeSplitArrays` to ensure splitTVS works.

### External Pre-conditions

N/A

### Attack Path

1. User attempts to split a TVS
2. They get 2 TVS' which don't add up to the original, excluding fees

### Impact

- NFT owners receive **less than their intended allocation** for each split.
- The total sum of splits will **not equal the original allocation**, resulting in **lost tokens**.
- Severity: **High**, as this directly impacts funds distribution and user balances.

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_merge_TVS -vvvv
```

First, 3 different bugs needs to be fixed, so in `FeesManager::calculateFeeAndNewAmountForOneTVS` fix the following code:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+ newAmounts = new uint256[](length);
+ for (uint256 i; i < length; i++) {
- for (uint256 i; i < length;) {
```

Then in `_computeSplitArrays`, fix the following code:

```diff
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;

+       alloc.amounts = new uint256[](nbOfFlows);
+       alloc.vestingPeriods = new uint256[](nbOfFlows);
+       alloc.vestingStartTimes = new uint256[](nbOfFlows);
+       alloc.claimedSeconds = new uint256[](nbOfFlows);
+       alloc.claimedFlows = new bool[](nbOfFlows);
```

Now when running the test, you will see in the logs that `splitTVS` with 50/50 will split a TVS of amount `5e17` into `2.5e17` and `1.25e17`, resulting in a `1.25e17` loss!
```md
├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 1, amounts: [250000000000000000 [2.5e17]], vestingPeriods: [3024000 [3.024e6]], vestingStartTimes: [1], claimedSeconds: [0])

├─ emit TVSSplit(projectId: 0, isBiddingProject: false, splitNftId: 1, nftId: 3, amounts: [125000000000000000 [1.25e17]], vestingPeriods: [3024000 [3.024e6]], vestingStartTimes: [1], claimedSeconds: [0])
```

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
        vm.prank(KOL1);
        vesting.claimRewardTVS(0);
        vm.startPrank(KOL2);
        vesting.claimRewardTVS(0);
        nft.transferFrom(KOL2, KOL1, 2);
        vm.stopPrank();
        vm.startPrank(KOL1);
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;
        vesting.splitTVS(0, percentages, 1);
    }
}
```

### Mitigation

- **Keep an immutable copy of the original allocation** before starting the loop:

```solidity
uint256[] memory originalAmounts = allocation.amounts; // keep original intact
```

- Pass this copy to `_computeSplitArrays` instead of reading from storage each time:

```solidity
Allocation memory alloc = _computeSplitArrays(originalAmounts, allocation, percentage, nbOfFlows);
```
  