# [000676] [M-2] Changing `vestingPeriodDivisor` invalidates existing bids and blocks updates
  
  ### Summary

The owner can change `vestingPeriodDivisor` at any time, but existing bidders must obey the new divisor when calling `updateBid`. If the new divisor is incompatible with their original vesting period, they can no longer increase their amount—even though the bid was valid when placed—effectively bricking their position and any planned top-ups.

### Root Cause

 `setVestingPeriodDivisor` immediately overwrites `vestingPeriodDivisor` (`AlignerzVesting.sol:397-405`) without migrating existing bids. Later, `updateBid` enforces `newVestingPeriod % vestingPeriodDivisor == 0` when `newVestingPeriod > 1` (lines 741-758). If the owner tightens the divisor (e.g., from 1 day to 30 days), all bidders whose `bid.vestingPeriod` is not a multiple of the new value cannot choose any `newVestingPeriod >= oldVestingPeriod` that satisfies the modulus, so they are stuck.

### Internal Pre-conditions

  1. Bidders already have `bids[msg.sender]` recorded with a `vestingPeriod` that is not a multiple of the new divisor.
  2. Owner calls `setVestingPeriodDivisor` with a new value.

### External Pre-conditions

None.

### Attack Path

  1. User places a bid with `vestingPeriod = 45 days` while divisor is 15 days (valid).
  2. Owner later sets `vestingPeriodDivisor = 30 days`.
  3. User attempts to increase their bid amount via `updateBid`. Because `newVestingPeriod` must be `>= old` and divisible by 30 days, no permissible value exists (45 % 30 != 0, 60 > old but still must be multiple, etc.). Requirement fails and the transaction reverts forever.

### Impact

 Legitimate bidders become unable to increase amounts or extend vesting, which can freeze capital and contradict governance expectations. While funds are not directly stolen, key functionality is bricked until a redeploy or manual migration.

### PoC

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";

contract GovernanceBricksBidTest is Test {
    AlignerzVesting private vesting;
    AlignerzNFT private nft;
    MockUSD private stable;
    Aligners26 private token;

    address private bidder = address(0xB1D);

    function setUp() public {
        vesting = new AlignerzVesting();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri");
        stable = new MockUSD();
        token = new Aligners26("Aligners", "A26Z");

        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));
        vesting.transferOwnership(address(this));

        stable.mint(bidder, 1_000 ether);
        vm.prank(bidder);
        stable.approve(address(vesting), type(uint256).max);

        vesting.launchBiddingProject(address(token), address(stable), block.timestamp, block.timestamp + 7 days, "0x0", false);
        vesting.setVestingPeriodDivisor(1 days);
        vm.prank(bidder);
        vesting.placeBid(0, 100 ether, 45 days);
    }

    function test_updateBidRevertsAfterDivisorTightening() public {
        vesting.setVestingPeriodDivisor(30 days);

        vm.prank(bidder);
        vm.expectRevert(); // new divisor incompatible with 45-day bid
        vesting.updateBid(0, 120 ether, 45 days);
    }
}
```

### Mitigation

 Either (a) forbid tightening the divisor once bids exist, (b) store each bid’s divisor and validate updates against the divisor in effect at bid creation, or (c) allow grandfathered bids to bypass the modulus check (e.g., `if (newVestingPeriod == bid.vestingPeriod) {}`) so they can at least add funds without changing vesting.

  