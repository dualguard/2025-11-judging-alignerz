# [000765] `_computeSplitArrays` uses uninitialized memory arrays, causing all `splitTVS` calls to revert and making TVS splitting unusable
  
  

### Summary

The incorrect memory handling in `_computeSplitArrays` will cause an inability to split TVS allocations for NFT holders as any user calling `splitTVS` on a TVS with flows will always revert during split processing.

### Root Cause

In `AlignerzVesting.sol:1112-1140` the function `_computeSplitArrays` writes to dynamic arrays inside a memory `Allocation` struct without ever allocating those arrays, so any `splitTVS` call on a TVS with at least one flow will revert when trying to index into `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, etc.

```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc //@audit not initialized
        )
    {
        uint256[] memory baseAmounts = allocation.amounts; 
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
  >>  alloc.assignedPoolId = allocation.assignedPoolId;
 >>   alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

The dynamic arrays `alloc.amounts`, `alloc.vestingPeriods`, `alloc.vestingStartTimes`, `alloc.claimedSeconds`, and `alloc.claimedFlows` are never initialized with `new` in memory, so accessing `alloc.amounts[j]` and similar indexes into an uninitialized memory array, which triggers a runtime bounds check failure and reverts. As a result, every `splitTVS` call that reaches this path fails, and the split allocations are never created.

### Internal Pre-conditions

1. A TVS NFT must exist with at least one vesting flow, so that `allocation.amounts.length` (and `nbOfFlows`) is greater than zero for the NFT being split.
2. The holder of that NFT must be able to call `splitTVS(projectId, percentages, splitNftId)` with a valid `projectId`, `splitNftId` they own, and `percentages` that sum to `BASIS_POINT`, so that `_computeSplitArrays` is reached with `nbOfFlows > 0`.

### External Pre-conditions

1. No specific external protocol or oracle condition is required; the bug is purely in the vesting contract’s in-memory array handling.
2. The chain state only needs to allow normal ERC20 transfers and NFT minting so that users can obtain TVS NFTs and then attempt to split them.

### Attack Path

1. A user participates in a reward or bidding flow and ends up owning a TVS NFT with at least one vesting flow, so that `allocation.amounts.length > 0` for that NFT in `AlignerzVesting`.
2. The user calls `splitTVS(projectId, percentages, splitNftId)` with `percentages.length >= 1` and a `splitNftId` they own, passing percentages that sum to `BASIS_POINT`.
3. Inside `splitTVS`, the contract computes `nbOfFlows = allocation.amounts.length` and then calls `_computeSplitArrays(allocation, percentage, nbOfFlows)` for each new TVS that should be created.
4. `_computeSplitArrays` enters the `for (uint256 j; j < nbOfFlows;)` loop and, on the first iteration, tries to execute `alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;` while `alloc.amounts` is still an uninitialized dynamic memory array.
5. The Solidity runtime detects an out-of-bounds write on an uninitialized memory array and the transaction reverts, so the split never completes, and no split allocations are written to storage.
6. As a result, the `splitTVS` feature is unusable for any real TVS position with flows, blocking users from splitting their vesting positions.

### Impact

The TVS NFT holders cannot split their vesting positions at all whenever the TVS has at least one flow, because any attempt to call `splitTVS` reverts inside `_computeSplitArrays`, making the split functionality effectively broken for all real allocations.

### PoC

Please insert below test under `test` folder in `SplitTVSBug.t.sol` name and run with `forge test --mt test_splitTVS_reverts_due_to_uninitialized_split_arrays -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract SplitTVSBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;

    function setUp() public {
        owner = address(this);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting implementation = new AlignerzVesting();

        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(proxy));
        vesting.setTreasury(address(1));

        token.approve(address(vesting), type(uint256).max);
    }

    function test_splitTVS_reverts_due_to_uninitialized_split_arrays() public {
        address kol1 = makeAddr("kol1");

        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory kolTVS = new address[](1);
        kolTVS[0] = kol1;

        uint256[] memory TVSamounts = new uint256[](1);
        TVSamounts[0] = 100 ether;

        uint256 totalTVSAllocation = TVSamounts[0];
        uint256 vestingPeriod = 30 days;

        token.transfer(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(
            rewardProjectId,
            totalTVSAllocation,
            vestingPeriod,
            kolTVS,
            TVSamounts
        );

        vm.prank(kol1);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId1 = nft.getTotalMinted();

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.prank(kol1);
        vm.expectRevert();
        vesting.splitTVS(rewardProjectId, percentages, nftId1);
    }
}
```

Result:
```bash
Ran 1 test for test/SplitTVSBug.t.sol:SplitTVSBugTest
[PASS] test_splitTVS_reverts_due_to_uninitialized_split_arrays() (gas: 822019)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.67ms (533.58µs CPU time)

Ran 1 test suite in 85.06ms (1.67ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Mitigation

Inside `_computeSplitArrays`, allocate all the dynamic arrays in the memory `Allocation` struct before writing to them, so that the loop writes into correctly sized memory arrays that `_assignAllocation` can safely copy into storage.


  