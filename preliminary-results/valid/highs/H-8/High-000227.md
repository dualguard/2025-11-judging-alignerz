# [000227] Any unclaimed TVS flow will brick dividend setup for every TVS holder
  
  ### Summary

The missing loop increment in `getUnclaimedAmounts` will cause a permanent revert of `_setAmounts/_setDividends` for all TVS holders as any NFT owner with an unclaimed flow keeps the iterator stuck at `i = 0`, exhausting gas before the dividend snapshot can be written.

### Root Cause

In [protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:140-161](https://github.com/dualguard/2025-11-alignerz/blob/ed2b9764732b5cdc4b8b35209cd2263f72210f31/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) the loop uses `continue` statements ([lines 148-151](https://github.com/dualguard/2025-11-alignerz/blob/ed2b9764732b5cdc4b8b35209cd2263f72210f31/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L151)) before the only `unchecked { ++i; }` ([line 157](https://github.com/dualguard/2025-11-alignerz/blob/ed2b9764732b5cdc4b8b35209cd2263f72210f31/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L157)). Whenever `claimedSeconds[i] == 0` or `claimedFlows[i] == true`, `i` never increments and the function never completes, leaving `_setAmounts`/`_setDividends` permanently unusable.

### Internal Pre-conditions

1. Treasury or owner deploys `A26ZDividendDistributor` and mints at least one TVS NFT so that `_setAmounts()` iterates over it.
2. The NFT’s flow still has `claimedSeconds == 0` (the default state before any vesting claim) or `claimedFlows == true`.
3. The owner funds the distributor with USDC/USDT and calls `setUpTheDividends()` to start the snapshot.

### External Pre-conditions

None

### Attack Path

1. TVS holder mints or holds any NFT whose `claimedSeconds` array still contains zeroes (normal immediately after allocation).
2. Treasury calls `setUpTheDividends()` (or `_setAmounts()` directly) after depositing stablecoins.
3. While iterating inside `getUnclaimedAmounts`, the branch at line 149 executes, `continue` fires before `++i`, and the loop restarts with `i = 0`.
4. The loop never progresses, the call runs out of gas, and `_setAmounts()` cannot update `unclaimedAmountsIn` or `totalUnclaimedAmounts`, blocking `_setDividends()` forever.

### Impact

TVS holders cannot receive any dividend distribution because the owner cannot complete `_setAmounts/_setDividends`. The entire stablecoin balance becomes stuck inside `A26ZDividendDistributor` until the contract is patched.

### PoC

1. Create test file `A26ZDividendDistributor.t.sol` and place in `protocol/test/` folder.
2. Copy code below into `A26ZDividendDistributor.t.sol`.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity 0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract A26ZDividendDistributorTest is Test {
    A26ZDividendDistributor internal distributor;
    MockNFT internal nft;
    MockVesting internal vesting;
    MockUSD internal stablecoin;
    MockUSD internal tvsToken;
    MockUSD internal allocationToken;

    address internal tvsHolder = address(0xBEEF);

    function setUp() public {
        nft = new MockNFT();
        vesting = new MockVesting();
        stablecoin = new MockUSD();
        tvsToken = new MockUSD();
        allocationToken = new MockUSD();

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(tvsToken)
        );

        uint256 nftId = nft.mint(tvsHolder);
        vesting.configureSingleFlowAllocation(nftId, address(allocationToken), 100 ether, 30 days);

        // Fund the distributor so the setup path is the only failing component.
        stablecoin.transfer(address(distributor), 1_000_000);
    }

    function testSetUpDividendsAlwaysRevertsDueToMissingLoopIncrement() public {
        (bool success,) =
            address(distributor).call{gas: 200_000}(abi.encodeWithSelector(distributor.setUpTheDividends.selector));
        assertFalse(success, "setUpTheDividends unexpectedly succeeded");
    }
}

contract MockNFT is IAlignerzNFT {
    uint256 public totalMinted;
    mapping(uint256 => address) internal owners;

    function mint(address to) external returns (uint256) {
        uint256 tokenId = totalMinted;
        owners[tokenId] = to;
        totalMinted++;
        return tokenId;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        address owner = owners[tokenId];
        require(owner != address(0), "Invalid token");
        return owner;
    }

    function burn(uint256 tokenId) external {
        owners[tokenId] = address(0);
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

contract MockVesting is IAlignerzVesting {
    mapping(uint256 => Allocation) internal allocations;

    function configureSingleFlowAllocation(
        uint256 nftId,
        address token,
        uint256 amount,
        uint256 vestingPeriod
    ) external {
        Allocation storage alloc = allocations[nftId];
        alloc.token = IERC20(token);
        if (alloc.amounts.length == 0) {
            alloc.amounts.push(amount);
            alloc.vestingPeriods.push(vestingPeriod);
            alloc.vestingStartTimes.push(block.timestamp);
            alloc.claimedSeconds.push(0);
            alloc.claimedFlows.push(false);
        } else {
            alloc.amounts[0] = amount;
            alloc.vestingPeriods[0] = vestingPeriod;
            alloc.vestingStartTimes[0] = block.timestamp;
            alloc.claimedSeconds[0] = 0;
            alloc.claimedFlows[0] = false;
        }
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return allocations[nftId];
    }
}
```
3. Run `forge test --match-test testSetUpDividendsAlwaysRevertsDueToMissingLoopIncrement`

### Mitigation

Increment the loop counter before each `continue` branch (or rewrite the loop using a canonical `for (uint256 i = 0; i < len; ++i)`), ensuring every flow advances `i` exactly once so `_setAmounts()` can finish and dividends can be configured.
  