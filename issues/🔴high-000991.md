# [000991] Dividend distributor will cause denial of service for TVS holders [High]
  
  ### Summary

The inverted token filter in [A26ZDividendDistributor.sol#L141](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) will cause a denial of service for TVS holders as the dividend distributor will exclude matching tokens causing division by zero in dividend allocation.

### Root cause

In `A26ZDividendDistributor.sol#L141` the token filter logic is inverted, excluding NFTs whose underlying token matches the configured token instead of including them.

```solidity
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

### Internal pre-conditions

1. Owner needs to configure the dividend distributor with a token that matches the underlying token of existing TVS NFTs
2. TVS holders need to have minted NFTs that vest the same token as configured in the dividend distributor
3. Owner needs to fund the distributor with stablecoins for distribution

### External pre-conditions

None required.

### Attack path

1. Owner calls `setUpTheDividends()` to initiate dividend distribution
2. The system calls `_setAmounts()` which invokes `getTotalUnclaimedAmounts()`
3. For each NFT, `getUnclaimedAmounts()` returns 0 due to the inverted filter at line 141
4. `totalUnclaimedAmounts` becomes 0 as all NFTs are excluded
5. The system calls `_setDividends()` which attempts division by `totalUnclaimedAmounts` (0) at line 218
6. The transaction reverts due to division by zero, preventing dividend setup

### Impact

The TVS holders cannot receive their intended dividend distributions as the core functionality of the dividend module is completely blocked. The allocated stablecoins remain trapped in the distributor contract until the code is fixed or the configuration is changed to use a different token.

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
// no proxy needed for this PoC; deploy implementation directly and initialize

contract DividendDistributorDOS is Test {
    Aligners26 internal token;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;
    MockUSD internal stable;
    A26ZDividendDistributor internal distributor;

    address internal owner;

    function setUp() public {
        owner = address(this);

        // Deploy core protocol components
        token = new Aligners26("Aligners26", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        stable = new MockUSD();

        // Deploy AlignerzVesting implementation directly and initialize
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint/burn NFTs
        nft.addMinter(address(vesting));

        // Set up a Reward Project that vests the same token as configured in the distributor
        // start now, 1 day claim window
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 1 days);

        // Fund vesting with reward token and configure two KOL allocations
        uint256 tvsAmount1 = 1_000 ether;
        uint256 tvsAmount2 = 2_000 ether;
        token.approve(address(vesting), tvsAmount1 + tvsAmount2);
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        address[] memory kols = new address[](2);
        kols[0] = alice;
        kols[1] = bob;
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = tvsAmount1;
        amounts[1] = tvsAmount2;
        vesting.setTVSAllocation(0, tvsAmount1 + tvsAmount2, 90 days, kols, amounts);

        // Each KOL claims their TVS, minting 2 NFTs (ids 1 and 2)
        vm.prank(alice);
        vesting.claimRewardTVS(0);
        vm.prank(bob);
        vesting.claimRewardTVS(0);

        // Deploy dividend distributor configured for the SAME token as the NFTs' underlying token
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            30 days,
            address(token)
        );

        // Fund distributor with stablecoin to distribute
        stable.transfer(address(distributor), 10_000e6);
    }

    function test_DividendSetupRevertsOnMatchingToken() public {
        // With all NFTs matching configured token, getUnclaimedAmounts() returns 0 for each,
        // totalUnclaimedAmounts stays 0 and _setDividends() divides by zero -> revert.
        vm.expectRevert();
        distributor.setUpTheDividends();
    }
}
```

### Mitigation

Consider correcting the token filtering logic in `getUnclaimedAmounts()` to include NFTs that match the configured token rather than excluding them. The fix involves inverting the conditional check from equality to inequality. Additionally, consider implementing a safeguard in `_setDividends()` to handle edge cases where `totalUnclaimedAmounts` might be zero, preventing division by zero errors in unexpected scenarios.

  