# [000276] Nested loops and excessive storage access in `getTotalUnclaimedAmounts()` lead to excessive gas usage and eventual DoS
  
  ### Summary

`getTotalUnclaimedAmounts()` uses nested logic and loops to aggregate all unclaimed tokens across all TVS NFTs. As the number of TVSs grows, the function becomes extremely gas-heavy and will eventually exceed the block gas limit on the deployed chains, causing a DoS for this function and for other functions that rely on it.

### Root Cause

In `getTotalUnclaimedAmounts()`, the function iterates over all minted NFTs and for each NFT calls `getUnclaimedAmounts()` to compute its unclaimed amount:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136
```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
@>      for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
@>          if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

Inside [getUnclaimedAmounts()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161) ,  the gas usage is very high, mainly due to three parts:

1.	For the same `vesting.allocationOf()` it is called `6` times in a row. Each call is an external call with ABI decoding, and each creates new memory arrays:

```solidity
		if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
```

2. For each NFT, there is an inner for loop over every flow in that TVS, performing arithmetic on each flow:
```solidity
		for (uint i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
```

3.	Finally, it writes the aggregated result to storage:
```solidity
		unclaimedAmountsIn[nftId] = amount;
```
Because these steps are repeated for every single NFT in the system, the total gas usage grows linearly with the number of TVSs. As more reward projects and bidding projects are launched, and TVS NFT holders (KOLs and bidders) increase, the total number of NFTs will become large enough that `getTotalUnclaimedAmounts()` runs out of gas and exceeds the block gas limit, causing a DoS.

In other words, the function is inherently designed to be extremely gas-expensive, and it does not require a very large number of NFTs before it becomes unusable on chains like Arbitrum (~32M gas) or Polygon (~45M gas).

### Internal Pre-conditions

- There are many TVS NFTs minted (across reward projects and bidding projects), each with one or more vesting flows stored in the vesting contract.

### External Pre-conditions

- The gas used by `getTotalUnclaimedAmounts()` on the chain where the protocol is deployed exceeds the per-block gas limit.

### Attack Path

1.	AlignerZ launches around `15` reward or bidding projects, and each project distributes TVSs to roughly `100` KOLs or bidders. This results in about `1500` TVS NFTs (and can easily grow further over time).
2.	The owner calls `setAmounts()` or `setUpTheDividends(`) to set the dividend amounts or configure distributions.
3.	These functions internally trigger `getTotalUnclaimedAmounts()`, which in turn calls `getUnclaimedAmounts()` for each NFT and loops over all flows.
4.	The total gas used by this operation exceeds typical chain limits (e.g. >`32M` on Arbitrum or >`45M` on Polygon), so the transaction cannot be included in a block and always reverts.

### Impact

As the number of TVSs (vesting NFTs) grows, `getUnclaimedAmounts()` and all functions that depend on it become unusable in practice. The protocol can no longer query all unclaimed tokens for all TVSs, cannot set dividends for TVS holders, and cannot set the amounts needed for dividend distribution. This results in a denial of service for critical dividend functionality and severely impacts protocol availability.

### PoC

Because `getUnclaimedAmounts()` has another issue (ABI mismatch with `allocationOf()`), that bug must be fixed first to avoid immediate reverts and allow gas measurement.

First, add a dedicated getter in `AlignerzVesting.sol` to replace the auto-generated mapping getter for `allocationOf`:
```diff
+   function getAllocationOf(uint256 nftId) public view returns (Allocation memory) {
+       return allocationOf[nftId];
+   }
```

Then add the corresponding interface in `IAlignerzVesting.sol`:
```diff
+   function getAllocationOf(uint256 nftId) external view returns (Allocation memory);
```

Update `getUnclaimedAmounts()` in `A26ZDividendDistributor.sol` to use `getAllocationOf()` instead of the auto-generated getter:
```diff
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-       if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
-       uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
-       uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
-       uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
-       bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
-       uint256 len = vesting.allocationOf(nftId).amounts.length;
        
+       if (address(token) == address(vesting.getAllocationOf(nftId).token)) return 0;
+       uint256[] memory amounts = vesting.getAllocationOf(nftId).amounts;
+       uint256[] memory claimedSeconds = vesting.getAllocationOf(nftId).claimedSeconds;
+       uint256[] memory vestingPeriods = vesting.getAllocationOf(nftId).vestingPeriods;
+       bool[] memory claimedFlows = vesting.getAllocationOf(nftId).claimedFlows;
+       uint256 len = vesting.getAllocationOf(nftId).amounts.length;
        ...
    }
```
This ensures `getUnclaimedAmounts()` no longer reverts due to ABI mismatch, so we can observe the gas usage.

Now according to the scenario described in the **Attack Path** section, create a test file `PoC.t.sol` in the test folder with the following code and run with:

`forge test --mt test_GetTotalUnclaimedAmountsGasTooHigh  --force -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract PoC is Test {
    AlignerzVesting public vesting;
    A26ZDividendDistributor public distributor;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;

    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp, 
            block.timestamp + 1_000_000,
            address(token)
        );

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function test_GetTotalUnclaimedAmountsGasTooHigh() public {
        // Simulate multiple reward projects with many KOLs
        uint256 PROJECT_NUM = 15;
        uint256 KOLS_NUM = 100;
    
        for (uint j; j < PROJECT_NUM; j++) {
            _createRewardProjectWithTVSs(KOLS_NUM); 
        }

        // 15 projects * 100 KOLs = 1500 TVS NFTs
        assertEq(nft.getTotalMinted(), 1500, "total minted TVS NFTs should be 1500");

        // Measure gas usage of getTotalUnclaimedAmounts()
        uint256 gasBefore = gasleft();
        distributor.getTotalUnclaimedAmounts();
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;

        // Assert that gasUsed is very large (above typical per-tx gas limits)
        assertGt(gasUsed, 45_000_000, "gas used should exceed 45M");
    }

    function _createRewardProjectWithTVSs(uint256 kolsNumber) public returns(uint256 rewardProjectId) {
        // Assume each project distributes the same TVS amount to kolsNumber KOLs
        uint256 rewardPerKol = 100e18;
        uint256 totalTVSAllocation = kolsNumber * rewardPerKol;
        uint256 vestingPeriod = 30 days;

        // Prepare KOL addresses and amounts
        address[] memory kols = new address[](kolsNumber);
        uint256[] memory amounts = new uint256[](kolsNumber);

        for (uint i; i < kolsNumber; i++) {
            kols[i] = makeAddr(string.concat("kol", vm.toString(i)));
            amounts[i] = 100e18;
        }

        // Project creator launches reward project and sets TVS allocations
        vm.startPrank(projectCreator);
        rewardProjectId = vesting.rewardProjectCount();
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 30 days);

        token.approve(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(rewardProjectId, totalTVSAllocation, vestingPeriod, kols, amounts);
        vm.stopPrank();

        // Each KOL claims their TVS NFT
        for (uint i; i < kolsNumber; i++) {
            vm.startPrank(kols[i]);
            vesting.claimRewardTVS(rewardProjectId);
            vm.stopPrank();
        }
    }
}
```

### Mitigation

The implementation should be redesigned to avoid scanning all TVS NFTs and all flows in a single call. Consider the following mitigations:
- In `getUnclaimedAmounts()`, call into vesting only once to fetch the Allocation struct, cache it in memory, and then read all fields from this single struct instead of calling `allocationOf()` multiple times.
- Reduce loop nesting and avoid iterating over all NFTs in a single transaction. If possible, consider optimizing the underlying NFT and accounting data structures.
  