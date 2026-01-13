# [000252] Incorrect token comparison in `A26ZDividendDistributor::getUnclaimedAmounts` causes skipped NFTs and dividend distribution DoS
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218

A single line logic error in `A26ZDividendDistributor::getUnclaimedAmounts` causes the function to return zero for NFTs whose allocation token actually matches the dividend distributor's token. As a result, the aggregate `totalUnclaimedAmounts` can become zero and subsequent dividend allocation divides by that zero value, reverting and blocking dividend distribution. This leads to a denial of service on dividend setup and funds that cannot be allocated.

### Root Cause

The comparison in `A26ZDividendDistributor::getUnclaimedAmounts` is inverted: it returns 0 when `address(token) == address(vesting.allocationOf(nftId).token)` instead of skipping NFTs that do not match. There is also no check before performing division by `totalUnclaimedAmounts`, so a zero total will cause a division by zero revert.

This can be backed by the protocol team response that shows the NFTs token should match the token tracked by the distributor:

<img width="838" height="677" alt="Image" src="https://github.com/user-attachments/assets/fec48ce8-7de1-4809-aac8-90ffdf936674" />

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Expected state: most or all NFTs token matches the token tracked by the distributor.
- Owner calls `setUpTheDividends`
- `getTotalUnclaimedAmounts` iterates NFTs and calls `getUnclaimedAmounts` for each.
- Because of the inverted comparison, `getUnclaimedAmounts` returns 0 for matching NFTs, so the sum `totalUnclaimedAmounts` becomes 0.
- `_setDividends` then executes:
```solidity
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)
```
causing a division by zero revert.

### Impact

- Dividend setup and allocation fail; owner cannot compute or set dividends.
- `stablecoinAmountToDistribute` becomes effectively locked until the bug is fixed because the allocation routine cannot complete.
- Users and integrators will see failures and funds held in contract, causing loss of confidence.

### PoC

Create a test file in `protocol/test/` folder and paste the below test in it. Run test with `forge test --match-test test_setUpTheDividends_reverts_due_to_bug -vvvv`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

contract MockVesting is IAlignerzVesting {
    IERC20 public tvs;
    constructor(address _tvs) {
        tvs = IERC20(_tvs);
    }

    function allocationOf(uint256) external view returns (Allocation memory alloc) {
        // Build a single-flow allocation with token == tvs
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1e6;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 1000;
    uint256[] memory vestingStartTimes = new uint256[](1);
    vestingStartTimes[0] = block.timestamp;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 0;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        alloc.amounts = amounts;
        alloc.vestingPeriods = vestingPeriods;
        alloc.vestingStartTimes = vestingStartTimes;
        alloc.claimedSeconds = claimedSeconds;
        alloc.claimedFlows = claimedFlows;
        alloc.isClaimed = false;
        alloc.token = tvs;
        alloc.assignedPoolId = 0;
    }
}

contract MockNFT is IAlignerzNFT {
    address public ownerAddress;
    uint256 public total;

    constructor(address _owner, uint256 _total) {
        ownerAddress = _owner;
        total = _total;
    }

    function mint(address) external pure returns (uint256) { revert(); }
    function extOwnerOf(uint256) external view returns (address) { return ownerAddress; }
    function burn(uint256) external pure { revert(); }
    function getTotalMinted() external view returns (uint256) { return total; }
}

contract PoC_GetUnclaimedAmounts is Test {
    MockUSD stable;
    MockUSD tvsToken;
    MockVesting vesting;
    MockNFT nft;
    A26ZDividendDistributor dist;

    function setUp() public {
        stable = new MockUSD();
        tvsToken = new MockUSD();
        // deploy mocks
        vesting = new MockVesting(address(tvsToken));
        nft = new MockNFT(address(this), 1);

        dist = new A26ZDividendDistributor(address(vesting), address(nft), address(stable), block.timestamp, 1000, address(tvsToken));

        // fund distributor with stablecoins
        stable.mint(address(this), 1_000_000 * 10 ** stable.decimals());
        stable.transfer(address(dist), 500_000 * 10 ** stable.decimals());
    }

    function test_setUpTheDividends_reverts_due_to_bug() public {
        // Because of the inverted token check in getUnclaimedAmounts, totalUnclaimedAmounts becomes zero
        // and _setDividends performs a division by zero, causing the call to revert.
        vm.expectRevert();
        dist.setUpTheDividends();
    }
}
```

### Mitigation

## Mitigation

Fix the logic:
Replace the inverted comparison so the function only returns 0 for NFTs that do NOT match the distributor token:
```solidity
if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  