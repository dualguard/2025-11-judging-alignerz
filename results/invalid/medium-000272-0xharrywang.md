# [000272] `distributeRewardTVS()` will fail when some KOLs have already claimed
  
  ### Summary

`distributeRewardTVS()` iterates over all KOLs for a reward project and calls `_claimRewardTVS()` for each of them. If any KOL has already claimed their reward earlier, `_claimRewardTVS()` will revert for that KOL. This single revert stops the whole loop and prevents `distributeRewardTVS()` from completing, causing a DoS for the project-wide distribution.

### Root Cause

In `distributeRewardTVS()`, the function loops over all KOLs and calls `_claimRewardTVS()`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535
```solidity
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

Inside `_claimRewardTVS()`, there is a strict check on the KOL’s TVS reward. If the stored reward amount is greater than `0`, it is set to `0` and the claim continues. If the amount is already `0`, the function reverts:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L601-L623
```solidity
    function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        uint256 amount = rewardProject.kolTVSRewards[kol];
@>      require(amount > 0, Caller_Has_No_TVS_Allocation());
@>      rewardProject.kolTVSRewards[kol] = 0;
		...
    }
```

However, `_claimRewardTVS()` is also called by `claimRewardTVS()`. If a KOL calls `claimRewardTVS()` before `distributeRewardTVS()` is executed, their `rewardProject.kolTVSRewards[kol]` is set to `0`.

Later, when `distributeRewardTVS()` is called and the loop reaches this KOL, `_claimRewardTVS() sees amount == 0` and reverts with `Caller_Has_No_TVS_Allocation()`. This single revert aborts the whole batch distribution for the reward project and makes `distributeRewardTVS()` unusable in that state.

### Internal Pre-conditions

- There is a reward project with at least one KOL that has a non-zero TVS allocation, and at least one KOL has already claimed their TVS via `claimRewardTVS()`, so their stored TVS reward amount is now zero.
- The current time has passed the project’s claim deadline (`block.timestamp > rewardProject.claimDeadline`), making `distributeRewardTVS()` callable.

### External Pre-conditions

_No response_

### Attack Path

1.	The protocol launches a reward project and sets KOL TVS allocations using `launchRewardProject()` followed by `setTVSAllocation()`.
2.	Before `distributeRewardTVS()` is executed, one KOL calls `claimRewardTVS()` (either normally or by front-running), which sets their `kolTVSRewards` entry to `0`.
3.	After `block.timestamp > rewardProject.claimDeadline`, someone calls `distributeRewardTVS()` for that `rewardProjectId`. When the loop reaches the KOL who already claimed, `_claimRewardTVS() sees amount == 0` and reverts with `Caller_Has_No_TVS_Allocation()`, causing the entire `distributeRewardTVS()` call to revert and making it unusable for that reward project.

### Impact

Any KOL can, intentionally or unintentionally, prevent `distributeRewardTVS()` from working. Once this happens, any attempt to run `distributeRewardTVS()` for that reward project will fail, causing a DoS of the batch distribution function and preventing the remaining TVS allocations from being reliably distributed.

### PoC

The PoC will demonstrate the scenario described in the **Attack Path** section.

To run the PoC, create a test file `PoC.t.sol` in the test folder with the following code and run with:

`forge test --mt test_DistributeRewardTVSRevertsIfAnyKolClaimed --force -vvvv`

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

contract PoC is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

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

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_DistributeRewardTVSRevertsIfAnyKolClaimed() public {
        // Reward project and KOLs info
        uint256 rewardProjectId = 0;

        address kol1 = makeAddr("kol1");
        address kol2 = makeAddr("kol2");
        address kol3 = makeAddr("kol3");

        // Arbitrary external account (should not be allowed to distribute)
        address anyone = makeAddr("anyone");
        
        // Launch reward project
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1_000_000);

        uint256 totalTVSAllocation = 10000e18;
        uint256 vestingPeriod = 30 days;
        address[] memory kols = new address[](3);
        kols[0] = kol1;
        kols[1] = kol2;
        kols[2] = kol3;

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 2000e18;
        amounts[1] = 3000e18;
        amounts[2] = 5000e18;

        // Set TVS allocations for KOLs
        token.approve(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(rewardProjectId, totalTVSAllocation, vestingPeriod, kols, amounts);
        vm.stopPrank();
        
        // KOL1 claims before distributeRewardTVS()
        vm.prank(kol1);
        vesting.claimRewardTVS(rewardProjectId);
        
        // After claim deadline, distributeRewardTVS() will revert with Caller_Has_No_TVS_Allocation error
        vm.warp(block.timestamp + 30 days);

        vm.expectRevert(AlignerzVesting.Caller_Has_No_TVS_Allocation.selector);
        vesting.distributeRewardTVS(rewardProjectId, kols);
    }
}
```

Output:
```shell
    ├─ [3978] ERC1967Proxy::fallback(0, [0x093d5d1908FBfaE1Dd3D2B21eBAbFb4451188D51, 0x4Eb93708e2B993936d233919e79C49E8e026dFd9, 0x6DA39581043310Ff1FF53B159d76fCdDc8735D17])
    │   ├─ [3219] AlignerzVesting::distributeRewardTVS(0, [0x093d5d1908FBfaE1Dd3D2B21eBAbFb4451188D51, 0x4Eb93708e2B993936d233919e79C49E8e026dFd9, 0x6DA39581043310Ff1FF53B159d76fCdDc8735D17]) [delegatecall]
    │   │   └─ ← [Revert] Caller_Has_No_TVS_Allocation()
    │   └─ ← [Revert] Caller_Has_No_TVS_Allocation()
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.74s (377.67µs CPU time)
```


### Mitigation

Modify `distributeRewardTVS()` logic so it skips KOLs that have already claimed instead of reverting for them.
  