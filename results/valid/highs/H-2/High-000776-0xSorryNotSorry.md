# [000776] mergeTVS and splitTVS will always revert
  
  ### Summary

The missing initialization of the `newAmounts` array in the fee helper will cause a complete denial of service for merge and split operations for all TVS holders as any user who calls `mergeTVS` or `splitTVS` will always revert during the fee calculation step.

### Root Cause

In `FeesManager.sol:L169-L173` the function `calculateFeeAndNewAmountForOneTVS` returns a dynamic array `newAmounts` but never allocates it, so its length is zero and the first write `newAmounts[0] = ...` is an out-of-bounds access that always reverts.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
>>      newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

This helper is called from both merge and split logic in `AlignerzVesting.sol`, so any attempt to use those flows will hit this revert before any state change:

AlignerzVesting.sol:L1010-L1013:

```solidity
uint256[] memory amounts = mergedTVS.amounts;
uint256 nbOfFlows = mergedTVS.amounts.length;
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
mergedTVS.amounts = newAmounts;
```

AlignerzVesting.sol:L1066-L1070:

```solidity
uint256[] memory amounts = allocation.amounts;
uint256 nbOfFlows = allocation.amounts.length;
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
allocation.amounts = newAmounts;
token.safeTransfer(treasury, feeAmount);
```

### Internal Pre-conditions

1. The owner configures a project and creates at least one TVS allocation such that `allocation.amounts.length > 0` for some NFT (through the normal reward or bidding flows).  
2. The owner sets `splitFeeRate` and/or `mergeFeeRate` to valid values so that `calculateFeeAndNewAmountForOneTVS` is actually called with `length > 0` when users call `splitTVS` or `mergeTVS`.

### External Pre-conditions

1. No special external protocol conditions are required; normal ERC20 behavior for the vested token and the stablecoin is enough to reach the revert in the local fee calculation logic.

### Attack Path

1. The admin configures the system (launches a project, allocates TVS, sets fees), and users obtain NFTs with TVS allocations where `amounts.length > 0`.  
2. A user who owns such an NFT calls `splitTVS` with any valid `percentages` array; `splitTVS` loads `allocation.amounts`, computes `nbOfFlows > 0`, and calls `calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows)`.  
3. Inside `calculateFeeAndNewAmountForOneTVS`, the loop enters with `i = 0`, `length > 0`, and `newAmounts.length == 0`, and the first write `newAmounts[0] = ...` triggers a `panic: array out-of-bounds access (0x32)`, reverting the whole call.  
4. The same happens for `mergeTVS`: it calls `calculateFeeAndNewAmountForOneTVS(mergeFeeRate, mergedTVS.amounts, nbOfFlows)`, which immediately reverts on the first write to `newAmounts[0]`, so the merge never proceeds.

### Impact

The TVS holders cannot use the `mergeTVS` and `splitTVS` features at all, resulting in a full denial of service of these vesting management operations, while no funds are directly stolen but the protocol’s intended UX and position management flexibility are broken.

### PoC

Please add below as `FeesManagerBug.t.sol` in the `test` folder and run with `forge test -vvvv --match-path test/FeesManagerBug.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract FeesManagerBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public kol;

    uint256 constant REWARD_PROJECT_ID = 0;

    function setUp() public {
        owner = address(this);
        kol = makeAddr("kol");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        token.approve(address(vesting), type(uint256).max);
    }

    function _createRewardTVSForKol() internal returns (uint256 nftId) {
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory kolTVS = new address[](1);
        kolTVS[0] = kol;

        uint256[] memory TVSamounts = new uint256[](1);
        TVSamounts[0] = 100 ether;

        vesting.setTVSAllocation(REWARD_PROJECT_ID, TVSamounts[0], 30 days, kolTVS, TVSamounts);

        vm.prank(kol);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);
        nftId = nft.getTotalMinted();
    }

    function test_splitTVS_reverts_due_to_uninitialized_newAmounts_array() public {
        uint256 nftId = _createRewardTVSForKol();

        vesting.setSplitFeeRate(100);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.prank(kol);
        vm.expectRevert();
        vesting.splitTVS(REWARD_PROJECT_ID, percentages, nftId);
    }

    function test_mergeTVS_reverts_due_to_uninitialized_newAmounts_array() public {
        uint256 nftId = _createRewardTVSForKol();

        vesting.setMergeFeeRate(100);

        uint256[] memory projectIds = new uint256[](0);
        uint256[] memory nftIds = new uint256[](0);

        vm.prank(kol);
        vm.expectRevert();
        vesting.mergeTVS(REWARD_PROJECT_ID, nftId, projectIds, nftIds);
    }
}
```

<img width="949" height="1045" alt="Image" src="https://github.com/user-attachments/assets/ad4ce734-0357-4ce5-abb4-194cfb453010" />

### Mitigation

The helper should allocate the `newAmounts` array. A minimal fix is:
```diff

  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+        newAmounts = new uint256[](length);
          for (uint256 i; i < length;) {
              feeAmount += calculateFeeAmount(feeRate, amounts[i]);
              newAmounts[i] = amounts[i] - feeAmount;
              unchecked {
                   ++i;
             }
         }
    }
```

  