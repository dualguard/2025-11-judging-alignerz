# [000273] `distributeRewardTVS()` and `distributeStablecoinAllocation()` lack `onlyOwner` modifier, allowing anyone to distribute
  
  ### Summary

`distributeRewardTVS()` and `distributeStablecoinAllocation()` in `AlignerzVesting` lack the `onlyOwner` modifier. This contradicts the NatSpec description and the fully open access breaks the intended protocol design.

### Root Cause

`distributeRewardTVS()` and `distributeStablecoinAllocation()` lack the `onlyOwner` modifier.

However, in the NatSpec, it is explicitly stated that these functions are intended to be called by the owner:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L522-L535
```solidity
@>  /// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
@>  function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
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

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L537-L550
```solidity
@>  /// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
    /// @param rewardProjectId Id of the rewardProject
    /// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
@>  function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

This issue breaks the intended protocol design, because as long as the current time is greater than `rewardProject.claimDeadline`, these functions are usable by anyone, not just the `owner`.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.	The owner calls `launchRewardProject()` to launch a reward project and then calls `setTVSAllocation()` to set TVS allocations for KOLs.
2.	Until the claim deadline, some KOLs do not call `claimRewardTVS()`. According to the intended design, these unclaimed tokens for this `rewardProjectId` should no longer be distributed to KOLs after the deadline.
3.	However, once `block.timestamp > rewardProject.claimDeadline`, any user can call `distributeRewardTVS()` and force these expired allocations to be distributed to the KOLs who have not claimed yet.

### Impact

Because anyone can trigger these functions instead of only the owner deciding whether and when to distribute, the intended protocol design is broken.

### PoC

The PoC shows that anyone can call `distributeRewardTVS()` to perform the distribution. To run the PoC, create a `PoC.t.sol` file, put the code below into it, and run:

`forge test --mt test_DistributeRewardTVSLacksOnlyOwner --force -vvv`:

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

    function test_DistributeRewardTVSLacksOnlyOwner() public {
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
        
        // After claim deadline, anyone can call distributeRewardTVS() to distribute
        vm.warp(block.timestamp + 30 days);
        vm.prank(anyone);
        vesting.distributeRewardTVS(rewardProjectId, kols);
    }
}
```

### Mitigation

According to the NatSpec, add the `onlyOwner` modifier to these functions:

```diff
+   function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external onlyOwner {
-   function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
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
 
+   function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external onlyOwner {   
-   function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

  