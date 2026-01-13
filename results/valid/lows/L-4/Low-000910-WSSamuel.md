# [000910] Unsafe Assumption of ERC721Receiver Support Causes Full Distribution Failure
  
  ### Summary

`distributeRemainingRewardTVS()` uses **safe minting** to send NFTs to each KOL. If any KOL address does **not** support `IERC721Receiver` (EOA, contract without the interface, or malicious reverting contract), the mint operation **reverts**, halting the entire distribution loop.

### Root Cause

Inside [`distributeRemainingRewardTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L562):

```solidity
uint256 nftId = nftContract.mint(kol); // safe mint
```

- Safe minting requires the recipient to implement `onERC721Received`, and the contract **does not verify** whether the KOL supports ERC721 before minting.
- The loop iterates over all stored `kolTVSAddresses`, so _any_ non-compliant KOL causes a full function revert.

### Internal Pre-conditions

- At least one entry in `kolTVSAddresses` is an address that cannot receive ERC721 tokens.
- Owner calls `distributeRemainingRewardTVS()`.

### External Pre-conditions

N/A

### Attack Path

1. Owner launches a reward project and sets a TVS Allocation for a list of KOLs
2. There is a single KOL which doesn't support onERC721Received
3. Owner attempts to distribute to all KOL's via `distributeRemainingRewardTVS`, resulting in a revert

### Impact

- Entire reward distribution transaction **reverts**.
- All remaining KOL reward NFTs become **permanently trapped** unless the owner uses other functions.
- Combined with the separately reported bug in `distributeRewardTVS()` (array-length / out-of-bounds DoS), the owner has **no reliable way** to distribute remaining TVS rewards.

### PoC

Add the following test suite into `test/RewardProject.t.sol` and run the test

```solidity
forge clean
forge test --mt test_owner_attempts_to_distribute -vvvv
```

You will see that it will revert with a `TransferToNonERC721ReceiverImplementer()`, halting distribution:

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

contract MaliciousKOL {
// This contract does NOT implement ERC721 or any other standard
}

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public distributor;
    address public owner;
    address public projectCreator;
    address[] public bidders;
    MaliciousKOL public KOL2;

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
        KOL2 = new MaliciousKOL();
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

    function test_owner_attempts_to_distribute() public {
        address KOL1 = makeAddr("KOL1");
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 1e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolTVS = new address[](2);
        kolTVS[0] = KOL1;
        kolTVS[1] = address(KOL2);
        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 5e17;
        TVSamounts[1] = 5e17;
        vesting.setTVSAllocation(0, 1e18, 5 weeks, kolTVS, TVSamounts);
        vm.warp(2 weeks);
        vm.startPrank(projectCreator);
        vesting.distributeRewardTVS(0, kolTVS);
    }
}
```

### Mitigation

Before minting, check whether the KOL supports ERC721 receiving (since you pop, you will need to rearrange the `kolTVSAddresses`):

```solidity
if (!_supportsERC721Receiver(kol)) {
    // skip or handle separately
    continue;
}
```
  