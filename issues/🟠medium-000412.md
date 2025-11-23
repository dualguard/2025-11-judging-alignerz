# [000412] Duplicate KOL entries overwrite rewards, leaving earlier allocations stuck
  
  `setTVSAllocation` and `setStablecoinAllocation` push every address into the pending arrays and set rewards by address key. If the same KOL appears twice, the mapping value (kolTVSRewards/kolStablecoinRewards) and index are overwritten by the last occurrence, but `totalAmount` sums all entries and the contract pulls the full `total*Allocation` tokens in.

As a result, the claim path and index bookkeeping only expose the last duplicated amount. Earlier amounts remain in the contract, unclaimable. Array indices also become inconsistent because `kol*IndexOf` stores only the last entry.

## Impact
I'd rate the severity a medium

Impact: A portion of the owner-supplied allocation can be permanently stranded in the contract (funds directly at risk for the project), and claim bookkeeping is corrupted. No direct theft, but real loss of availability of deposited tokens.

Likelihood: It's an `OnlyOwner` function, but duplicate addresses can easily slip into off-chain allocation lists and no special conditions or exotic tokens needed. This is likely enough to occur in normal ops.

## POC
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {Options} from "openzeppelin-foundry-upgrades/Options.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

// PoC: duplicate KOL addresses overwrite earlier rewards, leaving part of the deposited allocation stuck.
contract AlignerzDuplicateKOLAllocationPoC is Test {
    AlignerzVesting private vesting;
    Aligners26 private token;
    AlignerzNFT private nft;
    MockUSD private usdt;

    address private kol;

    function setUp() public {
        kol = makeAddr("kol");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        Options memory opts;
        opts.unsafeSkipAllChecks = true;

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft))),
                opts
            )
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        token.approve(address(vesting), type(uint256).max);
        usdt.approve(address(vesting), type(uint256).max);
    }

    function test_DuplicateTVSAllocationStrandsExcessTokens() public {
        uint256 firstAmount = 10 ether;
        uint256 secondAmount = 20 ether;
        uint256 total = firstAmount + secondAmount;
        uint256 vestingPeriod = 30 days;
        uint256 startTime = block.timestamp;

        vesting.launchRewardProject(address(token), address(usdt), startTime, 7 days);

        address[] memory recipients = new address[](2);
        recipients[0] = kol;
        recipients[1] = kol; // duplicate entry

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = firstAmount;
        amounts[1] = secondAmount;

        vesting.setTVSAllocation(0, total, vestingPeriod, recipients, amounts);

        assertEq(token.balanceOf(address(vesting)), total, "full TVS deposit recorded");

        vm.prank(kol);
        vesting.claimRewardTVS(0);
        uint256 nftId = nft.getTotalMinted();

        vm.warp(startTime + vestingPeriod + 1);
        vm.prank(kol);
        vesting.claimTokens(0, nftId);

        // Only the last duplicate entry is honored; the first allocation is stuck in the contract.
        assertEq(token.balanceOf(kol), secondAmount, "only last duplicate allocation is claimable");
        assertEq(token.balanceOf(address(vesting)), firstAmount, "earlier allocation remains locked");
    }

    function test_DuplicateStablecoinAllocationLeavesLockedBalance() public {
        uint256 firstAmount = 50e6;
        uint256 secondAmount = 70e6;
        uint256 total = firstAmount + secondAmount;

        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);

        address[] memory recipients = new address[](2);
        recipients[0] = kol;
        recipients[1] = kol; // duplicate entry

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = firstAmount;
        amounts[1] = secondAmount;

        vesting.setStablecoinAllocation(0, total, recipients, amounts);

        assertEq(usdt.balanceOf(address(vesting)), total, "full stablecoin deposit recorded");

        vm.prank(kol);
        vesting.claimStablecoinAllocation(0);

        // Only the last duplicate entry is honored; the first allocation stays trapped.
        assertEq(usdt.balanceOf(kol), secondAmount, "only last duplicate allocation paid out");
        assertEq(usdt.balanceOf(address(vesting)), firstAmount, "earlier allocation stuck in contract");
    }
}

```

## Recommendation
Reject duplicates up front (track seen addresses in a temp set and revert on repeat), or accumulate amounts per unique address before writing arrays/mappings, ensuring one entry per KOL and aligning the transferred total with claimable amounts.
  