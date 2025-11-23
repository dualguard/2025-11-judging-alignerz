# [000989] Users will suffer permanent loss of dividend entitlements through premature claims [Medium]
  
  
### Summary

The missing check in [A26ZDividendDistributor.sol:189-201](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L189-L201) will cause a permanent loss of dividend entitlements for TVS holders as users will call `claimDividends()` after the vesting start time but before dividend allocation, causing vested time to burn without receiving any corresponding payout.

### Root cause

In `A26ZDividendDistributor.sol:198` the `claimDividends()` function increments `claimedSeconds` regardless of whether any dividend liability exists for the user, allowing vesting time to be permanently consumed when `dividendsOf[user].amount` is zero.

### Internal pre-conditions

1. Owner needs to set `startTime` to be in the past relative to current block timestamp
2. `dividendsOf[user].amount` needs to be exactly `0` (no dividends allocated yet)
3. User needs to have a valid TVS position that will eventually receive dividend allocation

### External pre-conditions

None required.

### Attack path

1. User calls `claimDividends()` after `startTime` but before the owner calls `setUpTheDividends()` or `setDividends()`
2. The function calculates `secondsPassed = block.timestamp - startTime` and updates `dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds)` even though `totalAmount` is zero
3. Owner later allocates dividends through `_setDividends()` which sets `dividendsOf[user].amount` based on unclaimed TVS tokens
4. User calls `claimDividends()` again but receives reduced payout proportional to remaining vesting time after subtracting the previously burned `claimedSeconds`

### Impact

The TVS holders suffer an approximate loss proportional to the time elapsed between their premature claim and dividend allocation. The burned portion of vesting time becomes permanently unrecoverable, with the corresponding stablecoins remaining stranded in the contract and withdrawable by the owner via `withdrawStuckTokens()`. This represents direct user fund loss and compromises the fairness of the dividend distribution mechanism.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test} from "forge-std/Test.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "../../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../../src/MockUSD.sol";

contract DividendDistributor_ClaimBurnsTime_PoC is Test {
    Aligners26 internal token; // project token
    MockUSD internal stable;   // dividend stablecoin
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;
    A26ZDividendDistributor internal distributor;

    address internal owner = address(this);
    address internal user = address(0xBEEF);

    uint256 internal constant TVS_AMOUNT = 100 ether; // 100 tokens vested in TVS
    uint256 internal constant DIST_VESTING_PERIOD = 1000; // seconds

    uint256 internal rpStart;

    function setUp() public {
        // Deploy core components
        token = new Aligners26("Aligners26", "A26Z");
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "ALGN", "ipfs://base/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint/burn NFTs
        nft.addMinter(address(vesting));

        // Launch a reward project with start time T0 and claim window
        rpStart = block.timestamp + 1; // start shortly in the future
        vesting.launchRewardProject(address(token), address(stable), rpStart, 30 days);

        // Fund vesting with project tokens and set a KOL TVS allocation
        token.approve(address(vesting), type(uint256).max);
        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TVS_AMOUNT;
        vesting.setTVSAllocation(0, TVS_AMOUNT, 180 days, kols, amounts);

        // User claims the reward TVS -> mints NFT id 1 to user
        vm.prank(user);
        vesting.claimRewardTVS(0);

        // Mint a dummy NFT (id 2) via the vesting (minter) to an EOA so that
        // nft.getTotalMinted() >= 2, ensuring the distributor's 0..len-1 loops
        // include tokenId 1.
        vm.prank(address(vesting));
        nft.mint(address(0xCAFE));

        // Deploy the dividend distributor with distribution vesting period
        // Pass a token address that is different from the TVS token so the
        // snapshot includes our TVS (getUnclaimedAmounts != 0).
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            rpStart,
            DIST_VESTING_PERIOD,
            address(0xDEAD)
        );
    }

    function test_PoC_ClaimBurnsTimeBeforeAllocation() public {
        // 1) After startTime but BEFORE dividends are set, user calls claimDividends.
        // Advance 400 seconds into the distribution vesting window.
        vm.warp(rpStart + 400);
        vm.prank(user);
        distributor.claimDividends(); // amount == 0, but claimedSeconds becomes 400

        // Also have the user claim a small portion of TVS tokens to ensure
        // vesting.claimedSeconds > 0 to avoid an infinite loop in the
        // distributor's getUnclaimedAmounts() implementation when snapshotting.
        vm.prank(user);
        vesting.claimTokens(0, 1);

        // 2) Fund the distributor and set up dividends afterwards.
        // Fund 1,000 units of stablecoin (6 decimals -> 1_000e6)
        uint256 fund = 1_000 * 1e6;
        stable.mint(address(this), fund);
        stable.transfer(address(distributor), fund);

        // Owner allocates dividends snapshot now
        // Simulate the owner allocating dividends at a later time by
        // directly writing to the distributor's dividendsOf[user].amount
        // mapping storage using vm.store. This mimics setUpTheDividends()
        // semantics without relying on its snapshot helpers.
        //
        // Storage slot layout (taking Ownable's _owner at slot 0 into account):
        //  1: startTime
        //  2: vestingPeriod
        //  3: totalUnclaimedAmounts
        //  4: stablecoinAmountToDistribute
        //  5: vesting
        //  6: nft
        //  7: stablecoin
        //  8: token
        //  9: dividendsOf (mapping address => Dividend{amount, claimedSeconds})
        // 10: unclaimedAmountsIn (mapping)
        bytes32 base = keccak256(abi.encode(user, uint256(9)));
        // set dividendsOf[user].amount = fund
        vm.store(address(distributor), base, bytes32(uint256(fund)));

        // 3) Move to the end of the distribution vesting period and claim once.
        // Expected behavior (without bug): user would receive 100% of 'fund'.
        // Actual due to bug: only (1000 - 400) / 1000 = 60% is paid; 40% is lost.
        vm.warp(rpStart + DIST_VESTING_PERIOD);

        uint256 userBalBefore = stable.balanceOf(user);
        vm.prank(user);
        distributor.claimDividends();
        uint256 userBalAfter = stable.balanceOf(user);

        uint256 expectedPaid = (fund * 600) / 1000; // 60%
        assertEq(userBalAfter - userBalBefore, expectedPaid, "User did not receive the reduced amount (60%)");

        // The remaining 40% stays stranded in the distributor and can be withdrawn by owner.
        uint256 leftover = stable.balanceOf(address(distributor));
        assertEq(leftover, fund - expectedPaid, "Leftover should equal burned 40% of dividends");

        uint256 ownerBalBefore = stable.balanceOf(owner);
        distributor.withdrawStuckTokens(address(stable), leftover);
        uint256 ownerBalAfter = stable.balanceOf(owner);
        assertEq(ownerBalAfter - ownerBalBefore, leftover, "Owner should be able to extract stranded funds");
    }
}
```

### Mitigation

Consider implementing a check in `claimDividends()` to ensure dividend allocation has occurred before processing any claims. One approach could be to revert when `dividendsOf[user].amount` is zero, requiring users to wait until dividends are properly allocated. Alternatively, the function could defer time tracking until the first successful claim with a non-zero amount, ensuring that vesting time is only consumed when actual payouts occur.
  