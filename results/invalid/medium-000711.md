# [000711] Post-deadline distribution loops underflow when no recipients
  
  
## summary
Both “distribute remaining” functions seed the loop with `len - 1` even when the list is empty, causing an immediate underflow revert.

## Finding Description
In `AlignerzVesting`, the owner-only distribution functions compute `uint256 i = len - 1` unconditionally before a `for` loop guarded by the array length. When `len == 0`, the subtraction underflows.

[affected function code](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L554-L576)
```solidity
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
        address kol = rewardProject.kolTVSAddresses[i];
        ...
        rewardProject.kolTVSAddresses.pop();
        emit RewardTVSDistributed(rewardProjectId, kol, nftId, amount, vestingPeriod);
        unchecked { --i; }
    }
}
```
Root cause: `len - 1` evaluated when `len == 0`. Highest-impact scenario: after claim deadlines, if all KOLs already claimed, calling either function reverts, breaking batch post-deadline operations and scripts.

Locations:
- [`protocol/src/contracts/vesting/AlignerzVesting.sol:558`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L558) (TVS)
-[ `protocol/src/contracts/vesting/AlignerzVesting.sol:585`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L585) (stablecoin)

## Impact
Medium. Owner maintenance tasks can be bricked when lists are empty.
## poc
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract AlignerzVestingUnderflowPOC is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    function setUp() public {
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

        // Launch a reward project; leave KOL lists empty
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
    }

    function test_DistributeRemainingRewardTVS_EmptyList_Underflow() public {
        vm.warp(block.timestamp + 2);
        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingRewardTVS(0);
    }

    function test_DistributeRemainingStablecoin_EmptyList_Underflow() public {
        vm.warp(block.timestamp + 2);
        vm.expectRevert(stdError.arithmeticError);
        vesting.distributeRemainingStablecoinAllocation(0);
    }
}
```
## Recommendation
Guard `len > 0` or derive the index inside the loop. Simple guard:


```diff
uint256 len = rewardProject.kolTVSAddresses.length;
- for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
+ require(len > 0, "No pending TVS distributions");
+ for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
    ...
}
```
Alternatively rewrite as:
```solidity
for (; rewardProject.kolTVSAddresses.length > 0;) {
    uint256 i = rewardProject.kolTVSAddresses.length - 1;
    ...
}
```

  