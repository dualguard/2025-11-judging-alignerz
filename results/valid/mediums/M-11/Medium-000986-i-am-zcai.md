# [000986] Multiple dividend allocations will cause underpayment of users and stuck funds [Medium]
  
  
### Summary

The backdating of newly allocated dividends to the original vesting start time will cause an underpayment of users and stuck funds as multiple calls to `setUpTheDividends()` during an active vesting period will incorrectly apply historical vesting time to new dividend amounts.

### Root cause

In [A26ZDividendDistributor.sol:218](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218) the dividend amounts are accumulated using the `+=` operator without adjusting for the time when new allocations are made, causing new dividends to inherit previously elapsed vesting time.

### Internal pre-conditions

1. Owner needs to call `setUpTheDividends()` to set initial dividend allocations
2. Users need to claim partial dividends before additional allocations are made
3. Owner needs to call `setUpTheDividends()` again during the active vesting period to add more dividend funds

### External pre-conditions

None required.

### Attack path

1. Owner calls `setUpTheDividends()` to establish initial dividend allocations for users
2. Users call `claimDividends()` to claim partial dividends during the vesting period, updating their `claimedSeconds`
3. Owner calls `setUpTheDividends()` again mid-vesting to add additional dividend funds
4. The `_setDividends()` function adds new amounts to existing `dividendsOf[owner].amount` without resetting or adjusting `claimedSeconds`
5. Users call `claimDividends()` for subsequent claims, receiving payments calculated on the inflated total amount but using the same historical `claimedSeconds`

### Impact

The users suffer an approximate loss equal to the difference between intended linear vesting payouts and the time-weighted average calculation. The protocol suffers permanent loss of funds as the underpaid amounts become stuck in the distributor contract. In the demonstrated scenario, users receive 1,450 USDC instead of the expected 1,900 USDC from their allocations, leaving 450 USDC permanently inaccessible.

### POC

`protocol/test/poc/DividendTopUpBackdating.t.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {MockUSD} from "../../src/MockUSD.sol";
import {A26ZDividendDistributor} from "../../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../../src/interfaces/IAlignerzVesting.sol";
// Note: avoid using the upgrades helper to prevent full-build validation requirements in this PoC

contract DividendTopUpBackdatingTest is Test {
    Aligners26 internal token;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;
    MockUSD internal usdc;
    A26ZDividendDistributor internal dist;

    address internal owner;
    address internal alice;

    // Constants
    uint256 internal constant VESTING_PERIOD = 100 days;

    // Storage layout helper: Ownable places _owner at slot 0, so the distributor's
    // own variables start at slot 1. Therefore dividendsOf is at slot 9.
    function _dividendsSlot(address user) internal pure returns (bytes32) {
        return keccak256(abi.encode(user, uint256(9)));
    }

    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");

        // Deploy core contracts
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        usdc = new MockUSD();

        // Deploy vesting implementation directly and initialize
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint NFTs
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(0xBEEF));

        // Fund vesting with project tokens for rewards
        uint256 totalTVS = 1_000 ether; // arbitrary TVS amount (unrelated to stablecoin dividends sizing)
        token.approve(address(vesting), totalTVS);

        // Launch a reward project with immediate start
        vesting.launchRewardProject(address(token), address(usdc), block.timestamp, 30 days);

        // Set a single KOL TVS allocation for alice
        address[] memory kols = new address[](1);
        uint256[] memory amounts = new uint256[](1);
        kols[0] = alice;
        amounts[0] = totalTVS;
        vesting.setTVSAllocation(0, totalTVS, 365 days, kols, amounts);

        // Alice claims her TVS -> mints NFT id 1 owned by alice
        vm.prank(alice);
        vesting.claimRewardTVS(0);

        // Advance a bit and have Alice claim some vested tokens so that
        // allocation.claimedSeconds[0] > 0, avoiding an infinite loop bug in
        // the distributor's getUnclaimedAmounts when claimedSeconds == 0.
        vm.warp(block.timestamp + 1 days);
        vm.prank(alice);
        vesting.claimTokens(0, 1);

        // Proceed without additional sanity checks to keep the PoC focused on the distributor bug

        // Note: we avoid calling splitTVS since it triggers internal array writes that
        // are not required for this PoC and may introduce unrelated panics.

        // Mint a dummy NFT (id 2) to an EOA so that distributor iteration [0..getTotalMinted()-1]
        // includes tokenId 1 (since ERC721A starts at 1). Minting to a contract would require
        // ERC721Receiver; mint to alice instead.
        nft.mint(alice);

        // Deploy dividend distributor (we won't use snapshot helpers to avoid unrelated loops)
        dist = new A26ZDividendDistributor(address(vesting), address(nft), address(usdc), block.timestamp, VESTING_PERIOD, address(0));

        // Fund the distributor with the initial 1000 USDC
        usdc.transfer(address(dist), 1_000_000);

        // Simulate an initial dividend allocation of 1000 USDC to alice by directly
        // setting dividendsOf[alice].amount. This mirrors the effect of setDividends()
        // without invoking the snapshot code which has independent issues.
        bytes32 slot = _dividendsSlot(alice);
        vm.store(address(dist), slot, bytes32(uint256(1_000_000))); // amount
        uint256 dbgAmt = uint256(vm.load(address(dist), slot));
        assertEq(dbgAmt, 1_000_000, "dbg: initial amount not set");
        vm.store(address(dist), bytes32(uint256(slot) + 1), bytes32(0)); // claimedSeconds
    }

    function test_BackdatingAndUnderpaymentAcrossTopUp() public {
        // Halfway through dividend vesting
        vm.warp(block.timestamp + VESTING_PERIOD / 2); // +50 days

        // Alice claims: expects 1000 * 50/100 = 500 USDC
        uint256 balBefore = usdc.balanceOf(alice);
        vm.prank(alice);
        dist.claimDividends();
        uint256 claim1 = usdc.balanceOf(alice) - balBefore;
        assertEq(claim1, 500_000, "initial half-vesting claim should be 500 USDC");

        // Top up mid-vesting: add 900 USDC more and simulate the protocol adding
        // new dividends to the same user by increasing dividendsOf[user].amount
        // by +900k (mirror of += in _setDividends). We do not change claimedSeconds.
        usdc.transfer(address(dist), 900_000); // fund the contract
        bytes32 base = _dividendsSlot(alice);
        uint256 current = uint256(vm.load(address(dist), base));
        vm.store(address(dist), base, bytes32(current + 900_000));

        // Advance by 25 days (to 75% of vesting period)
        vm.warp(block.timestamp + 25 days);

        // Alice claims again. With backdating, claim is computed from the full current
        // amount (1,900 USDC), yielding 1,900 * 25% = 475 USDC for this slice.
        balBefore = usdc.balanceOf(alice);
        vm.prank(alice);
        dist.claimDividends();
        uint256 claim2 = usdc.balanceOf(alice) - balBefore;
        assertEq(claim2, 475_000, "second slice computed on inflated totalAmount (1,900 * 25%)");

        // Advance to vest end and claim final slice
        vm.warp(block.timestamp + 25 days);
        balBefore = usdc.balanceOf(alice);
        vm.prank(alice);
        dist.claimDividends();
        uint256 claim3 = usdc.balanceOf(alice) - balBefore;
        assertEq(claim3, 475_000, "final slice computed on inflated totalAmount");

        // Totals: paid = 500 + 475 + 475 = 1,450 USDC, but deposits = 1,900 USDC
        // => 450 USDC remain stuck in the distributor due to incorrect accounting with backdated top-ups
        uint256 remaining = usdc.balanceOf(address(dist));
        assertEq(remaining, 450_000, "underpayment leaves funds stuck in distributor");

        // Alice received exactly the paid amounts
        assertEq(usdc.balanceOf(alice), 1_450_000, "alice should have received 1,450 USDC total");

        // Additionally, show the mid-vesting expected payment vs actual to highlight backdating:
        // Expected tranche-based mid-claim at 75%:
        // - Original tranche: 1000 * 25% = 250
        // - New tranche (at 50% start): 900 * (25% of period elapsed since top-up) = 225
        // => Expected 475 USDC, which matches the backdated result here; however,
        // the underpayment manifests at vest end, leaving 450k stuck.
    }
}
```

### Mitigation

Consider implementing per-tranche dividend tracking to properly account for allocations made at different times during the vesting period. One approach could be to store an array of dividend tranches, each containing the allocation amount, assignment timestamp, and claimed seconds specific to that tranche. When calculating claimable amounts, iterate through each tranche and compute vesting progress from its individual assignment time. Alternatively, consider modifying the settlement logic to snapshot the total claimed amount in USD terms when new dividends are added, then calculate future claims based on the incremental vesting of both existing and new allocations from their respective start times.
  