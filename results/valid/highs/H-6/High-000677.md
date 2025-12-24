# [000677] [H-6] Dividend setup loops over total minted supply and becomes permanently unusable
  
  ### Summary

Both `getTotalUnclaimedAmounts` and `_setDividends` iterate over every NFT ID from `0..nft.getTotalMinted()-1`. As supply grows, these unbounded loops exceed the block gas limit, so `setUpTheDividends` / `setDividends` / `setAmounts` revert forever, freezing dividend configuration and trapping the stablecoin balance.

### Root Cause

In `A26ZDividendDistributor.sol:118-135` and `214-221`, the loops have no batching or pagination—they simply iterate `for (uint i; i < len;)` where `len = nft.getTotalMinted()`. Once `len` is large (tens of thousands), the gas needed to call `vesting.allocationOf` and update `dividendsOf` per NFT surpasses block limits.

### Internal Pre-conditions

  1. Many NFTs have been minted (normal growth of the protocol).
  2. Owner attempts to run dividend setup after enough NFTs exist.

### External Pre-conditions

None.

### Attack Path

1. Protocol launches several projects, minting a large number of vesting NFTs.
  2. Owner calls `setUpTheDividends`.
  3. `getTotalUnclaimedAmounts` loops through the entire minted range; gas usage increases linearly and eventually reverts once the supply exceeds what a single transaction can handle.
  4. Even if `_setAmounts` succeeds, `_setDividends` repeats the same full-scan loop and reverts as well. Dividend setup becomes permanently impossible.

### Impact

Dividend configuration DoS. Stablecoin deposits sit idle, and holders never receive dividends because the owner cannot complete the setup transaction. Severity is High since it bricks a core protocol module once NFT supply grows.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";

contract DividendUnboundedLoopTest is Test {
    A26ZDividendDistributor private dist;
    HeavyVesting private vesting;
    MockMassNFT private nft;
    MockUSD private stable;

    address private constant HOLDER = address(0xA11CE);

    function setUp() public {
        stable = new MockUSD();
        vesting = new HeavyVesting();
        nft = new MockMassNFT(HOLDER);
        dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            30 days,
            address(0xCAFE)
        );
    }

    function test_setUpTheDividends_runsOutOfGasWhenSupplyGrows() public {
        nft.setTotalMinted(1);
        stable.mint(address(dist), 100e6);
        uint256 gasBefore = gasleft();
        dist.setUpTheDividends();
        uint256 gasUsedSmall = gasBefore - gasleft();

        uint256 gasBudget = gasUsedSmall * 20;
        bytes memory callData = abi.encodeWithSelector(dist.setUpTheDividends.selector);

        stable.mint(address(dist), 100e6);
        (bool smallSuccess,) = address(dist).call{gas: gasBudget}(callData);
        assertTrue(smallSuccess, "small supply should fit within calibrated gas budget");

        nft.setTotalMinted(50_000);
        stable.mint(address(dist), 100e6);
        (bool largeSuccess,) = address(dist).call{gas: gasBudget}(callData);
        assertFalse(largeSuccess, "large supply exhausts the same gas budget");
    }
}

contract HeavyVesting is IAlignerzVesting {
    Allocation private alloc;

    constructor() {
        for (uint256 i; i < 8; ++i) {
            alloc.amounts.push(1e18);
            alloc.vestingPeriods.push(30 days);
            alloc.vestingStartTimes.push(block.timestamp);
            alloc.claimedSeconds.push(1); // avoid zero-claimed branch
            alloc.claimedFlows.push(false);
        }
        alloc.token = IERC20(address(0xDEAD));
    }

    function allocationOf(uint256) external view returns (Allocation memory) {
        return alloc;
    }
}

contract MockMassNFT is IAlignerzNFT {
    uint256 private totalMinted;
    address private ownerAddr;

    constructor(address owner_) {
        ownerAddr = owner_;
    }

    function setTotalMinted(uint256 newTotal) external {
        totalMinted = newTotal;
    }

    function mint(address to) external returns (uint256) {
        ownerAddr = to;
        totalMinted += 1;
        return totalMinted - 1;
    }

    function burn(uint256) external {}

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        require(tokenId < totalMinted, "nonexistent token");
        return ownerAddr;
    }
}
```

### Mitigation

Introduce pagination/batching (e.g., process subsets of NFTs per call), or maintain incremental accumulators updated on mint/burn events so dividend setup doesn’t require scanning the entire historical supply. Limit `len` per transaction to a manageable chunk and persist progress across calls.
  