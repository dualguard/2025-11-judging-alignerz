# [000990] Users will permanently lose dividend entitlements through zero-amount claims [High]
  
  
### Summary

The missing check in `A26ZDividendDistributor.sol:198` will cause a permanent loss of dividend entitlements for users as they will call `claimDividends()` after the vesting start time but before their dividend amounts are assigned, consuming vesting time without receiving any tokens.

### Root cause

In `A26ZDividendDistributor.sol#L198` there is a missing check that prevents updating `claimedSeconds` when the user's dividend amount is zero, allowing vesting time to be consumed without any corresponding token transfer.

### Internal pre-conditions

1. Admin needs to call the constructor to set `startTime` to be exactly the current block timestamp or earlier
2. User needs to have a TVS NFT with `dividendsOf[user].amount` to be exactly `0` (not yet assigned by admin)
3. Current block timestamp needs to be at least `startTime + 1` but less than `startTime + vestingPeriod`

### External pre-conditions

None required.

### Attack path

1. User calls `claimDividends()` after the vesting start time but before admin assigns their dividend amount, causing `claimedSeconds` to advance while `totalAmount` remains zero
2. Admin later assigns a positive dividend amount to the user through the setup process
3. User calls `claimDividends()` again, but only receives dividends proportional to the time elapsed since their previous zero-amount claim rather than the full elapsed time since vesting start

### Impact

The users suffer an approximate loss proportional to the time consumed by zero-amount claims. In the provided proof-of-concept, the user loses 700 out of 1000 units (70% of their intended dividend allocation), with the lost funds remaining permanently stranded in the distributor contract rather than being transferred to the rightful recipient.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "../../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

// PoC: Zero-amount dividend claims consume vesting time and reduce future payouts
contract DividendDistributor_ZeroAmountClaimPoC is Test {
    Aligners26 internal token;          // project token
    AlignerzNFT internal nft;           // TVS NFT
    AlignerzVesting internal vesting;   // vesting engine (UUPS proxy)
    MockUSD internal stable;            // dividend stablecoin
    A26ZDividendDistributor internal dist; // dividend distributor

    address internal projectOwner;
    address internal user;   // victim
    address internal other;  // second minter to satisfy distributor's off-by-one loop

    uint256 internal constant TVS_AMOUNT_USER = 100 ether;   // arbitrary units for TVS
    uint256 internal constant TVS_AMOUNT_OTHER = 200 ether;   // arbitrary units for TVS
    uint256 internal constant DISTRIBUTION = 1_000_000;       // 1,000 stablecoins with 6 decimals
    uint256 internal constant VESTING_PERIOD = 1000;          // seconds

    function setUp() public {
        projectOwner = makeAddr("projectOwner");
        user = makeAddr("user");
        other = makeAddr("other");

        // Deploy base contracts
        stable = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy vesting via UUPS proxy and grant minter
        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);
        nft.addMinter(proxy);

        // Hand over vesting ownership and fund project owner with tokens
        token.transfer(projectOwner, 1_000_000 ether);
        vesting.transferOwnership(projectOwner);

        // Launch a reward project and set allocations (so users can mint TVS NFTs)
        vm.startPrank(projectOwner);
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 7 days);
        token.approve(address(vesting), TVS_AMOUNT_USER + TVS_AMOUNT_OTHER);
        address[] memory kols = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        kols[0] = user; amounts[0] = TVS_AMOUNT_USER;
        kols[1] = other; amounts[1] = TVS_AMOUNT_OTHER;
        vesting.setTVSAllocation(0, TVS_AMOUNT_USER + TVS_AMOUNT_OTHER, 365 days, kols, amounts);
        vm.stopPrank();

        // Users claim TVS NFTs; ensure user's NFT is minted first (ID = 1)
        vm.prank(user);
        vesting.claimRewardTVS(0); // NFT ID 1 -> user

        vm.prank(other);
        vesting.claimRewardTVS(0); // NFT ID 2 -> other

        // Deploy the dividend distributor
        // Note: pass token address as address(0) so getUnclaimedAmounts() includes the NFTs (code excludes equal token)
        dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            VESTING_PERIOD,
            address(0)
        );

        // Fund distributor with stablecoins to distribute
        stable.transfer(address(dist), DISTRIBUTION);
    }

    function test_PoC_ZeroAmountClaimConsumesVestingTime() public {
        // Mark user's vesting flow as having some claimedSeconds so that
        // DividendDistributor's snapshot loop doesn't hit its 'continue' bug path.
        // Do a small token claim from vesting so claimedSeconds > 0 for NFT #1
        vm.warp(block.timestamp + 1);
        vm.prank(user);
        vesting.claimTokens(0, 1);

        // 1) After startTime but before dividends are set, the user calls claimDividends.
        //    This sets claimedSeconds even though amount == 0.
        vm.warp(block.timestamp + 400); // 400 seconds elapsed
        vm.prank(user);
        dist.claimDividends(); // transfers 0 but advances claimedSeconds to 400

        // 2) Admin sets dividends later. We simulate this by writing directly to the internal mapping
        //    via Foundry cheatcodes (equivalent effect of setUpTheDividends assigning amount).
        //    Storage layout (Ownable base consumes slot 0):
        //      1: startTime
        //      2: vestingPeriod
        //      3: totalUnclaimedAmounts
        //      4: stablecoinAmountToDistribute
        //      5: vesting
        //      6: nft
        //      7: stablecoin
        //      8: token
        //      9: dividendsOf (mapping)
        //    amount is at keccak256(abi.encode(user, uint256(9)))
        bytes32 slot = keccak256(abi.encode(user, uint256(9))); // base slot for dividendsOf[user]
        vm.store(address(dist), slot, bytes32(uint256(DISTRIBUTION)));
        // Sanity check: load back and ensure amount written
        bytes32 rawAmount = vm.load(address(dist), slot);
        assertEq(uint256(rawAmount), DISTRIBUTION, "amount not set");
        // Also check claimedSeconds after zero-amount claim is non-zero
        bytes32 rawClaimedSeconds = vm.load(address(dist), bytes32(uint256(slot) + 1));
        assertGt(uint256(rawClaimedSeconds), 0, "claimedSeconds should have advanced on zero-amount claim");

        // Sanity: total distribution available is 1,000e6 and entirely allocated to user
        // 3) Later, user claims again at t=700s; only (700 - 400) seconds are considered
        vm.warp(block.timestamp + 300); // now total elapsed since start = 700
        uint256 balBefore = stable.balanceOf(user);
        vm.prank(user);
        dist.claimDividends();
        uint256 balAfter = stable.balanceOf(user);

        // User received only 300/1000 of the total distribution instead of expected 700/1000
        assertEq(balAfter - balBefore, 300_000, "User should receive only 300 USDC due to time consumed by zero-claim");

        // The remaining 700 USDC is stranded in the distributor, demonstrating loss of entitlement
        assertEq(stable.balanceOf(address(dist)), 700_000, "700 USDC remains stranded in distributor");
    }
}
```

### Mitigation

Consider implementing protective logic to prevent vesting time consumption when no dividend amount is allocated. One approach could be adding a guard condition that skips the `claimedSeconds` update when `dividendsOf[user].amount` equals zero:

```diff
} else {
    secondsPassed = block.timestamp - startTime;
-   dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
+   if (totalAmount > 0) {
+       dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
+   }
}
```

Alternatively, consider implementing a mechanism that defers vesting time tracking until dividend amounts are properly assigned, ensuring that users cannot inadvertently consume their entitlement window before receiving allocations.
  