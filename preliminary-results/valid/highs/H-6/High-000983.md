# [000983] Protocol will experience denial of service for dividend distribution [medium]
  
  Protocol will experience denial of service for dividend distribution [medium]

### Summary

The unbounded loops in `getTotalUnclaimedAmounts()` and `_setDividends()` will cause a denial of service for dividend distribution as the protocol scales and the total minted NFTs will exceed gas limits during dividend setup operations.

### Root cause

In [A26ZDividendDistributor.sol:128-136](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128-L136) the `getTotalUnclaimedAmounts()` function uses unbounded iteration over all minted NFTs without pagination. In `A26ZDividendDistributor.sol:215-222` the `_setDividends()` function similarly iterates through all minted tokens using `nft.getTotalMinted()` as the loop bound, which grows continuously as new TVS NFTs are minted.

### Internal pre-conditions

1. NFT supply needs to grow from initial deployment to exceed gas limit threshold through normal minting operations

### External pre-conditions

None required.

### Attack path

1. Protocol operates normally and NFT supply grows through legitimate minting across different projects and participants
2. The owner calls `setUpTheDividends()` which internally calls `getTotalUnclaimedAmounts()` and `_setDividends()`
3. Both functions iterate through all minted NFTs using `nft.getTotalMinted()` as the upper bound
4. Each iteration performs external contract calls including `safeOwnerOf()` and `getUnclaimedAmounts()` increasing gas costs
5. Transaction reverts due to exceeding block gas limits when NFT count reaches the threshold

### Impact

The dividend distribution feature becomes unusable once the NFT supply exceeds the gas limit threshold. The owner can still withdraw deposited stablecoin funds but TVS holders cannot receive their intended dividend rewards through the normal mechanism. This breaks a key incentive system for token holders while core vesting functionality remains unaffected.

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

contract DividendSetupDoS is Test {
    Aligners26 token;
    AlignerzNFT nft;
    AlignerzVesting vesting;
    MockUSD stable;
    A26ZDividendDistributor distributor;

    address owner;

    function setUp() public {
        owner = address(this);

        // Deploy core contracts
        stable = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy UUPS proxy for vesting and set it as NFT minter
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        nft.addMinter(proxy);

        // Create a Reward Project with 1 KOL so at least one NFT has a real allocation
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 7 days);

        // Fund vesting with reward tokens (Aligners26) for allocations
        uint256 kolAmount = 1_000 ether;
        token.approve(address(vesting), kolAmount);
        address kol = makeAddr("kol");
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = kolAmount;
        vesting.setTVSAllocation(0, kolAmount, 90 days, kols, amounts);

        // KOL claims TVS -> mints 1 real TVS with allocation in vesting
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        // First minted NFT id should be 1 per ERC721A _startTokenId()
        assertEq(nft.ownerOf(1), kol);

        // Deploy the dividend distributor. Set `token` param different from the vested token
        // so getUnclaimedAmounts() includes the TVS allocation in the snapshot.
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            90 days,
            address(0) // different from Aligners26 to include allocations
        );

        // Fund the distributor with stablecoin for the dividend round
        stable.mint(address(distributor), 1_000_000); // 1 USDC (decimals=6) worth of rewards
    }

    function _mintManyEOA(uint256 count, uint256 seed) internal {
        // Test contract is a minter by default (set in AlignerzNFT constructor).
        for (uint256 i = 0; i < count; ++i) {
            address recipient = makeAddr(string.concat("user", vm.toString(seed + i)));
            nft.mint(recipient);
        }
    }

    function test_DividendSetup_DoS_by_UnboundedIteration() public {
        // Sanity: with very few NFTs, setup should succeed under a modest gas limit
        // Only 1 NFT exists from reward claim above.
        uint256 gasLimit = 2_000_000; // 2.0M gas budget
        (bool okSmall, ) = address(distributor).call{gas: gasLimit}(abi.encodeWithSignature("setUpTheDividends()"));
        assertTrue(okSmall, "setUpTheDividends should succeed with small NFT supply");

        // Now massively increase minted supply without allocations to simulate scale
        // This inflates nft.getTotalMinted() and forces the distributor's loops to iterate over all IDs.
        _mintManyEOA(12000, 1); // +12k NFTs minted to EOAs to amplify loop cost

        // Re-run with the SAME gas limit: expect failure due to out-of-gas from unbounded iteration
        (bool okLarge, ) = address(distributor).call{gas: gasLimit}(abi.encodeWithSignature("setUpTheDividends()"));
        assertFalse(okLarge, "setUpTheDividends unexpectedly succeeded under same gas after large minting");

        // Informational assertions: supply actually grew
        uint256 totalMinted = nft.getTotalMinted();
        assertGt(totalMinted, 1000, "Insufficient NFT supply minted for the DoS demonstration");
    }
}
```

### Mitigation

Consider implementing a batched approach for dividend setup operations to avoid gas limit constraints. One effective strategy would be to separate the dividend calculation and allocation phases into multiple transactions that process NFTs in manageable chunks. The implementation could introduce state variables to track processing progress across batches, allowing the system to resume from where it left off in subsequent transactions. This approach would maintain the current dividend distribution logic while ensuring the system remains functional regardless of NFT supply size.
  