# [000907] Incorrect Cumulative Fee Calculation Causes Over-Charging of Flows
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS()` accumulates `feeAmount` across iterations and subtracts the **cumulative fee** from each flow. This results in later flows to be charged the sum of fees for all previous flows plus their own, resulting in over-charging that grows with each indexed flow.

### Root Cause

In [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) core logic (simplified):

```solidity
for (uint256 i; i < length;) {
    feeAmount += calculateFeeAmount(feeRate, amounts[i]); // cumulative
    newAmounts[i] = amounts[i] - feeAmount;                // subtracts cumulative fee
}
```

Because `feeAmount` is cumulative, the `i`-th flow is reduced by the sum of fees for flows `0..i`, not by its own fee. Thus the second flow is charged (fee0 + fee1), the third charged (fee0 + fee1 + fee2), etc.

### Internal Pre-conditions

- Any call to `mergeTVS`, `splitTVS`, or any other logic that uses `calculateFeeAndNewAmountForOneTVS()` with `length > 1`.
- The 2 other bugs in this function are fixed, so it doesn't have a OOG or EVM revert (reported seperately).

### External Pre-conditions

N/A

### Attack Path

1. User has 2 TVS' he wishes to merge
2. User attempts to merge TVS, resulting in overcharging of fees

### Impact


- Users are systematically overcharged, especially for users with large flows.

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

Now when running the test, you will see the following logs, as you can see, the `feeAmount` is what should have been subtracted, however, `1.5e17` was taken from the amounts:

```md
Logs:
  Total Fees 1e17:
  New Amount 1 9.5e17:
  New Amount 2 9e17:
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

    function test_fee_manager_wrong_fees() public view {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1e18;
        amounts[1] = 1e18;
        //e 5%
        (uint256 feeAmount, uint256[] memory newAmounts) = vesting.calculateFeeAndNewAmountForOneTVS(500, amounts, 2);
        console.log("Total Fees %e:", feeAmount);
        console.log("New Amount 1 %e:", newAmounts[0]);
        console.log("New Amount 2 %e:", newAmounts[1]);
    }
}
```

### Mitigation

Charge each flow its own fee, not cumulative fees. Implement as follows.

### Corrected implementation (per-flow fee)

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 totalFeeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i = 0; i < length; ++i) {
        uint256 feeForFlow = calculateFeeAmount(feeRate, amounts[i]);
        totalFeeAmount += feeForFlow;
        newAmounts[i] = amounts[i] - feeForFlow;
    }
}
```

  