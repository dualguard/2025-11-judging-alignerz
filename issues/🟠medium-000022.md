# [000022] Error in the `kolTVSIndexOf` can cause users to not get their rewards if they do not claim before the deadline.
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L458C5-L477C6

when the function `setTVSAllocation` is called, the users address is appended to `kolTVSAddresses`  with `rewardProject.kolTVSAddresses.push(kol)`, however the rewards mapping to the array uses the index of the function argument 

```solidity
uint256 length = kolTVS.length;
        require(length == TVSamounts.length, Array_Lengths_Must_Match());
        uint256 totalAmount;
        for (uint256 i = 0; i < length; i++) {
            address kol = kolTVS[i];
            rewardProject.kolTVSAddresses.push(kol);
            uint256 amount = TVSamounts[i];
            rewardProject.kolTVSRewards[kol] = amount;
            // @audit this can override?
            rewardProject.kolTVSIndexOf[kol] = i;
            totalAmount += amount;
            emit TVSAllocated(rewardProjectId, kol, amount, vestingPeriod);
        }
```

this makes the `kolTVSIndexOf[kol]` not map correctly to `kolTVSAddresses`

after rewards are claimed the address is removed with 
```solidity
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
        address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
        rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
        rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
        rewardProject.kolTVSAddresses.pop();
```

but because the mapping is incorrect, legitimate users address are deleted and are unable to claim rewards

### Root Cause

duplicate address in `rewardProject.kolStablecoinAddresses`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. owner sets setTVSAllocation or setStablecoinAllocation
2. owner want to reset it because of an error or wants to add another set of users that contains a duplicate user already added
3. claim deadline has passed an some users have not claimed their rewards
4. duplicate address blocks the `function distributeRemainingRewardTVS` thereby locking users rewards

### Impact

It is possible for legitimate users to loose their rewards after the claim windows has passed

### PoC

```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {console} from "forge-std/console.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";


contract POC1 is Test {

    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public treasury;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;
    uint256 constant BID_FEE = 100_000;
    uint256 constant SPLIT_FEE = 200;

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
        treasury = makeAddr("treasury");
        
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        nft.addMinter(address(vesting));
        vesting.setTreasury(treasury);
        vesting.setVestingPeriodDivisor(1);
        vesting.setFees(
            BID_FEE,
            BID_FEE,
            SPLIT_FEE,
            SPLIT_FEE
        );

        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, BIDDER_USD);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_vul2() external {
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 2000);
        address[] memory kolTVS = new address[](3);
        kolTVS[0] = bidders[0];
        kolTVS[1] = bidders[1];
        kolTVS[2] = bidders[2];
        
        uint[] memory Amounts = new uint256[](3);
        Amounts[0] = 100e18;
        Amounts[1] = 100e18;
        Amounts[2] = 100e18;
        
        vesting.setTVSAllocation(0, 300e18 , 2000, kolTVS, Amounts);

        address extra_address_1 = makeAddr("extra_address_1");
        address extra_address_2 = makeAddr("extra_address_2");
        address extra_address_3 = makeAddr("extra_address_3");
        address extra_address_4 = makeAddr("extra_address_4");

        address[] memory extra_addresses = new address[](5);
        extra_addresses[0] = extra_address_1;
        extra_addresses[1] = extra_address_2;
        extra_addresses[2] = extra_address_3;
        extra_addresses[3] = bidders[0];
        extra_addresses[4] = extra_address_4;

        uint[] memory override_amounts = new uint256[](5);
        override_amounts[0] = 100e18;
        override_amounts[1] = 100e18;
        override_amounts[2] = 100e18;
        override_amounts[3] = 100e18;
        override_amounts[4] = 100e18;

        vesting.setTVSAllocation(0, 500e18, 2000, extra_addresses, override_amounts);
        
        vm.prank(bidders[0]);
        vesting.claimRewardTVS(0);

        vm.warp(block.timestamp + 2001);
        vesting.distributeRemainingRewardTVS(0);

        assert(token.balanceOf(extra_address_4) == 0);
    
        // fix prevent calling the function twice or reset the array instead of pushing
    }

}


```

### Mitigation

set `rewardProject.kolStablecoinAddresses` to `kolStablecoin` in `function setStablecoinAllocation` instead of appending to the array.
the same vulnurability is present in `setTVSAllocation`
  