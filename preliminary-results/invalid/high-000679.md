# [000679] [C-1] Dividend setup allocates the same funds multiple times and guarantees insolvency
  
  ### Summary

Every dividend setup run uses the distributor’s full stablecoin balance to compute new per-user dividend amounts, but already allocated-yet-unclaimed funds are never deducted. Running `setUpTheDividends` twice without depositing more tokens allocates the exact same money twice, so late claimants revert or receive nothing once the balance has been drained.

### Root Cause

In `A26ZDividendDistributor.sol:206-219`, `_setAmounts` copies the entire `stablecoin.balanceOf(address(this))` into `stablecoinAmountToDistribute` regardless of previously assigned dividends, and `_setDividends` then credits each owner with `+= unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts`. No variable tracks “already promised” funds, so previous allocations remain outstanding while the same tokens are promised again.

### Internal Pre-conditions

  1. At least one dividend round has been configured and users have not claimed their entire `dividendsOf[user].amount`.
  2. Owner re-runs `setUpTheDividends` (or `setAmounts` + `setDividends`) without first clearing prior allocations or zeroing `dividendsOf`.

### External Pre-conditions

None.

### Attack Path

  1. Owner deposits 100k stablecoin and calls `setUpTheDividends`. Each TVS holder is assigned a proportional amount totaling 100k.
  2. Before holders claim, owner calls `setUpTheDividends` again (perhaps to refresh `unclaimedAmountsIn`).
  3. `_setAmounts` again records `stablecoinAmountToDistribute = 100k` because the balance is unchanged.
  4. `_setDividends` adds another 100k-worth of entitlements to all holders. Aggregate `dividendsOf[*].amount` is now 200k while the contract still holds only 100k.
  5. Early claimants can withdraw until the 100k on-chain balance is exhausted; everyone else reverts due to insufficient funds.

### Impact

Guaranteed insolvency of the dividend pool whenever setup is run multiple times between user claims. Later claimants revert as the stablecoin balance cannot satisfy the inflated liabilities. Value at risk is the full dividend balance per setup cycle.

### PoC

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";

contract DividendDoubleAllocationTest is Test {
    A26ZDividendDistributor private dist;
    StaticVesting private vesting;
    MockOwnedNFT private nft;
    MockUSD private stable;
    address private constant ALICE = address(0xA11CE);
    uint256 private constant VESTING_PERIOD = 30 days;

    function setUp() public {
        stable = new MockUSD();
        vesting = new StaticVesting(1e18, 30 days);
        nft = new MockOwnedNFT();
        nft.setOwner(0, ALICE);
        dist = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stable),
            block.timestamp,
            VESTING_PERIOD,
            address(0xBEEF)
        );
    }

    function test_setUpTheDividends_doubleAllocationCausesInsolvency() public {
        uint256 deposit = 100e6; // 100 stablecoin units (6 decimals)
        stable.mint(address(dist), deposit);

        dist.setUpTheDividends();
        dist.setUpTheDividends(); // rerun without depositing more funds

        vm.warp(block.timestamp + VESTING_PERIOD + 1);
        vm.prank(ALICE);
        vm.expectRevert(); // transfer amount exceeds distributor balance
        dist.claimDividends();
    }
}

contract StaticVesting is IAlignerzVesting {
    Allocation private alloc;

    constructor(uint256 amount, uint256 vestingPeriod) {
        alloc.amounts.push(amount);
        alloc.vestingPeriods.push(vestingPeriod);
        alloc.vestingStartTimes.push(block.timestamp);
        alloc.claimedSeconds.push(1);
        alloc.claimedFlows.push(false);
        alloc.token = IERC20(address(0xCAFE));
    }

    function allocationOf(uint256) external view returns (Allocation memory) {
        return alloc;
    }
}

contract MockOwnedNFT {
    mapping(uint256 => address) private owners;
    uint256 private minted;

    function setOwner(uint256 tokenId, address owner) external {
        owners[tokenId] = owner;
        if (tokenId >= minted) {
            minted = tokenId + 1;
        }
    }

    function getTotalMinted() external view returns (uint256) {
        return minted;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        address owner = owners[tokenId];
        require(owner != address(0), "ERC721: invalid token ID");
        return owner;
    }
}

```

### Mitigation

Track and deduct previously allocated (but unclaimed) dividends when recomputing amounts, or zero/claim all outstanding dividends before running `setUpTheDividends` again. A simple fix is to reset every `dividendsOf[owner]` before the loop, or to maintain a “reserved” counter that is subtracted from the contract balance when calculating `stablecoinAmountToDistribute`.
  