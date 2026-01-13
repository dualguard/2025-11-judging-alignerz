# [000992] Infinite loop will cause denial of service for dividend setup operations [High]
  
  ### Summary


The missing index increment in early `continue` statements within `getUnclaimedAmounts()` will cause an infinite loop for dividend operations as the owner will encounter unbounded iteration when calling dividend setup functions.

### Root Cause

In [A26ZDividendDistributor.sol:148-151](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L150) the early `continue` statements skip the loop index increment, causing infinite loops when `claimedFlows[i]` is `true` or when `claimedSeconds[i]` equals `0`.

### Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` to set up dividend distribution
2. At least one TVS NFT needs to be minted and owned
3. TVS allocation needs to have `claimedSeconds[i]` to be exactly `0` (default initial state) or `claimedFlows[i]` to be exactly `true`

### External Pre-conditions

None required

### Attack Path

1. Owner calls `setUpTheDividends()` to initialize dividend distribution
2. Function calls `_setAmounts()` which calls `getTotalUnclaimedAmounts()`
3. `getTotalUnclaimedAmounts()` iterates through minted NFTs and calls `getUnclaimedAmounts(i)` for each owned NFT
4. `getUnclaimedAmounts()` enters the for-loop at line 147 and encounters a condition where `claimedFlows[i]` is `true` or `claimedSeconds[i]` equals `0`
5. The function executes `continue` at line 148 or 151, skipping the `unchecked { ++i; }` block and causing the same iteration to repeat indefinitely

### Impact

The dividend setup operations suffer complete denial of service, preventing the owner from initializing dividend distributions. The infinite loop causes gas exhaustion and transaction reversion, leaving `totalUnclaimedAmounts` and individual dividend allocations unset. While stablecoin reserves are not permanently locked due to the owner's ability to withdraw via `withdrawStuckTokens()`, the dividend distribution module becomes completely inoperable until the code is fixed or redeployed.

### PoC


```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test} from "forge-std/Test.sol";

// Real protocol contracts/interfaces
import {Aligners26} from "src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "src/MockUSD.sol";

contract DividendDoS_PoC is Test {
    Aligners26 internal token;            // Project token vested in TVS
    AlignerzNFT internal nft;             // TVS NFT
    AlignerzVesting internal vesting;     // Core vesting engine
    MockUSD internal stable;              // Stablecoin used by distributor
    A26ZDividendDistributor internal dist;// Vulnerable dividend distributor

    address internal admin = address(this);
    address internal kol; // KOL / TVS recipient (EOA)
    address internal kol2; // Second KOL to ensure total minted >= 2 (triggers iteration over id 1)

    function setUp() public {
        // Choose a non-contract EOA for KOL to avoid ERC721Receiver requirement
        kol = address(0xBEEF);
        kol2 = address(0xCAFE);

        // Deploy core contracts
        token = new Aligners26("Aligners26", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "A-NFT", "ipfs://base/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        // Allow vesting to mint/burn TVS NFTs
        nft.addMinter(address(vesting));

        // Deploy stablecoin and the vulnerable distributor
        stable = new MockUSD();

        // Note: pass a token different than the vested token so getUnclaimedAmounts() does NOT early-return 0
        // Using address(0) as a dummy token is enough to trigger the vulnerable path
        dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            90 days,
            address(0)
        );

        // Fund distributor with some stablecoin to make setup realistic
        stable.mint(address(dist), 1_000_000e6);

        // Launch a reward project that will mint a TVS with initial claimedSeconds == 0 (default)
        uint256 startTime = block.timestamp;
        vesting.launchRewardProject(address(token), address(stable), startTime, 30 days);

        // Allocate TVS to KOL and deposit tokens into vesting
        uint256 amount = 100 ether;
        token.approve(address(vesting), amount * 2);
        address[] memory kols = new address[](2);
        kols[0] = kol;
        kols[1] = kol2;
        uint256[] memory amts = new uint256[](2);
        amts[0] = amount;
        amts[1] = amount;
        vesting.setTVSAllocation(0, amount * 2, 60 days, kols, amts);

        // KOL claims reward TVS, creating allocation with claimedSeconds[0] = 0
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        vm.prank(kol2);
        vesting.claimRewardTVS(0);
    }

    // PoC: setUpTheDividends() enters an infinite loop (gas exhaustion) due to missing i++ on `continue` paths
    function test_DividendsSetup_RevertsDueToInfiniteLoop() public {
        // Expect any revert (out-of-gas / invalid execution), demonstrating DoS
        vm.expectRevert();
        dist.setUpTheDividends();
    }

    // Directly hitting the vulnerable function also reverts due to unbounded loop when claimedSeconds == 0
    function test_GetUnclaimedAmounts_RevertsDueToInfiniteLoop() public {
        // The first minted NFT ID in this ERC721A setup is 1
        vm.expectRevert();
        dist.getUnclaimedAmounts(1);
    }
}
```


### Mitigation

Consider restructuring the loop in `getUnclaimedAmounts()` to ensure the index is always incremented, regardless of which execution path is taken. One approach would be to move the increment logic outside the conditional blocks or use a standard for-loop structure with the increment in the loop declaration to prevent early `continue` statements from skipping the index increment.
  