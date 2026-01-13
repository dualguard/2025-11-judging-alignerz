# [000904] Duplicate KOL Entries Cause Loss of Allocation
  
  ### Summary

The `setTVSAllocation` function allows the owner to assign TVS allocations to several KOL addresses. However, if the input arrays `kolTVS` contain **duplicate addresses**, only the _last_ allocation for a given KOL is stored.

### Root Cause

[`setTVSAllocation`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L466-L469) writes KOL allocations using:

```solidity
rewardProject.kolTVSRewards[kol] = amount;
rewardProject.kolTVSIndexOf[kol] = i;
rewardProject.kolTVSAddresses.push(kol);
```

If `kolTVS` contains the same address twice:

```solidity
kolTVS = [A, A]
TVSamounts = [100, 200]
```

Then:

- First loop: `kolTVSRewards[A] = 100`
- Second loop: **overwrites** → `kolTVSRewards[A] = 200`

But `totalTVSAllocation` validation still checks:

```
total = 100 + 200 = 300
```

→ 100 tokens become unclaimable because the KOL can only claim the last written amount.

### Internal Pre-conditions

Admin needs to create a TVS allocation with duplicates

### External Pre-conditions

N/A

### Attack Path

1. Admin launches a reward project and sets a TVS Allocation with duplicates
2. User attempts to claim the TVS twice, but the 2nd one would revert as `rewardProject.kolStablecoinRewards[kol] = 0`
3. Furthermore, if the admin attempts to call `distributeRemainingRewardTVS` after the deadline has passed, it will mint the user a TVS with `amount=0`

### Impact

- **Loss of user funds**: Part of the allocated TVS becomes permanently unclaimable.
- **Unexpected behavior**: The owner may believe two allocations were created, but only one exists.

### PoC

Add the following test suite into `test/MergeTVS.t.sol` and run the test:

```solidity
forge clean
forge test --mt test_claim_2_tokens -vvvv
```

Now when running the test, you will see that the 2nd `claimRewardTVS` will revert, and when the owner calls `distributeRemainingRewardTVS`, the KOL will get minted a TVS with `amount = 0`.

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

    function test_claim_2_tokens() public {
        address KOL1 = makeAddr("KOL1");

        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(100);

        vm.startPrank(projectCreator);
        usdt.approve(address(vesting), 1e18);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 weeks);
        address[] memory kolTVS = new address[](2);
        kolTVS[0] = KOL1;
        kolTVS[1] = KOL1;
        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 5e17;
        TVSamounts[1] = 5e17;

        vesting.setTVSAllocation(0, 1e18, 5 weeks, kolTVS, TVSamounts);
        vm.stopPrank();
        vm.startPrank(KOL1);
        vesting.claimRewardTVS(0);
        vm.expectRevert();
        vesting.claimRewardTVS(0);
        vm.warp(5 weeks);
        vm.stopPrank();
        vm.startPrank(projectCreator);
        vesting.distributeRemainingRewardTVS(0);
    }
}
```

### Mitigation

Add a validation step to ensure there are no duplicate KOL addresses:

```solidity
mapping(address => bool) seen;

for (uint256 i; i < length; ++i) {
    address kol = kolTVS[i];
    require(!seen[kol], "Duplicate KOL address");
    seen[kol] = true;
}
```
  