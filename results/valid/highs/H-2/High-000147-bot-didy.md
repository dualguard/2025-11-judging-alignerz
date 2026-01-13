# [000147] Any user splitting will erase TVS allocations for holders
  
  ### Summary

Zero-length `newAmounts` allocation and cumulative fee subtraction in `calculateFeeAndNewAmountForOneTVS` will cause full loss/corruption of TVS vesting flows for NFT holders as any caller will invoke splitTVS/mergeTVS, hit `array out-of-bounds` or replace `allocations` with over-deducted/empty amounts.

### Root Cause

In FeesManager.sol (https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169), `calculateFeeAndNewAmountForOneTVS` returns an uninitialized `newAmounts` array and subtracts the running cumulative fee (`feeAmount`) from each flow instead of the per-flow fee, causing `array out-of-bounds` and wiping/over‑deducting allocations when `splitTVS`/`mergeTVS` use it.

### Internal Pre-conditions

1. Owner/vesting has already minted a TVS NFT with at least one vesting flow to a user (so `allocation.amounts.length > 0` for that `nftId`).
2. The NFT holder (attacker or honest user) retains ownership of that NFT, allowing them to call `splitTVS`/`mergeTVS` on it.

### External Pre-conditions

None required

### Attack Path

1. NFT holder calls `splitTVS` (or `mergeTVS`) on their vested NFT.
2. The call enters `calculateFeeAndNewAmountForOneTVS`, which tries to write into an uninitialized zero-length `newAmounts` array.
3. The write triggers an `array out-of-bounds` panic, causing the `split`/`merge` transaction to revert and the operation to fail.



### Impact

- The NFT holder cannot split or merge their TVS allocations (feature is fully DoS’d); vesting reconfiguration via `split`/`merge` is impossible.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";

contract FeeCalculationBugTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    MockUSD stable;

    address alice = makeAddr("alice");

    function setUp() public {
        // Deploy minimal stack (token/NFT/vesting proxy).
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri/");
        token = new Aligners26("Aligners", "ALN");
        stable = new MockUSD();

        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Allow vesting to mint NFTs and set a non-zero treasury to avoid distractions.
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Prepare a simple reward project with one TVS allocation to Alice.
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 30 days);
        token.approve(address(vesting), 1 ether);
        address[] memory kols = new address[](1);
        kols[0] = alice;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1 ether;
        vesting.setTVSAllocation(0, 1 ether, 30 days, kols, amounts);

        vm.prank(alice);
        vesting.claimRewardTVS(0); // Mints first NFT to Alice (token IDs start at 1).
    }

    function test_SplitTVSRevertsDueToUninitializedFeeArray() public {
        // Splitting triggers calculateFeeAndNewAmountForOneTVS, which writes into a zero-length array -> index OOB.
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5_000;
        percentages[1] = 5_000;

        uint256 mintedNftId = nft.totalSupply(); // First minted tokenId is 1 because ERC721A _startTokenId = 1.

        vm.prank(alice);
        vm.expectRevert(stdError.indexOOBError);
        vesting.splitTVS(0, percentages, mintedNftId);
    }
}
```

### Mitigation

Allocate `newAmounts = new uint256[](length);` before the loop in `calculateFeeAndNewAmountForOneTVS` (or inline allocate in callers) and subtract the per-flow fee: `uint256 fee = calculateFeeAmount(feeRate, amounts[i]); newAmounts[i] = amounts[i] - fee; feeAmount += fee;`.
  