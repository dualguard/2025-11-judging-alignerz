# [000909] Missing Array Initialization Causes mergeTVS and splitTVS to Always Revert
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` fails to initialize the `newAmounts` array, resulting in `newAmounts[i]` referencing **unallocated memory**, causing an immediate revert when written to. Both `mergeTVS` and `splitTVS` depend on this function, meaning **all merge and split operations fail**, leading to **complete loss of functionality** for TVS transformations.

### Root Cause

Inside [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174):

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
	for (uint256 i; i < length;) {
		feeAmount += calculateFeeAmount(feeRate, amounts[i]);
		newAmounts[i] = amounts[i] - feeAmount;
	}
}
```

Because `newAmounts` is never created:

- It remains the default value: **a zeroed memory pointer**.
- Any write such as:

```solidity
newAmounts[i] = amounts[i] - feeAmount;
```

results in a **memory write to an invalid location**, causing an immediate revert.

### Internal Pre-conditions

- User attempts to call `mergeTVS` or `splitTVS

### External Pre-conditions

N/A

### Attack Path

1. User has 2 TVS' he wishes to merge
2. User attempts to merge TVS, but it reverts
3. Same with TVS splitting

### Impact

- `mergeTVS` always reverts.
- `splitTVS` always reverts.
- Users cannot merge or split TVS positions at all.

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_merge_TVS -vvvv
```

You will see that mergeTVS will revert with `panic: array out-of-bounds access (0x32)`:

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
        vm.prank(KOL1);
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 0;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = 2;
        vesting.mergeTVS(0, 1, projectIds, nftIds);
    }
}
```

### Mitigation

Add the missing initialization:

```solidity
newAmounts = new uint256[](length);
```
  