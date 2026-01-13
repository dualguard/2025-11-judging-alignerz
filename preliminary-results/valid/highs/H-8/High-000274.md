# [000274] Missing index increment with `continue` in `getUnclaimedAmounts()` for loop causes out-of-gas
  
  ### Summary

`getUnclaimedAmounts()` uses a for loop to iterate over all flows of a TVS and uses continue to skip certain flows. However, whenever the loop hits a continue, the index `i` is not incremented. This makes the loop never progress past that flow and results in an infinite loop that reverts with out-of-gas, causing a DoS for this function and related functions.

### Root Cause

In `getUnclaimedAmounts()`, the function iterates over each flow of a TVS and tries to skip further calculation when a flow is already claimed or when the code decides to treat the flow as fully counted. This is done using `continue`:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
		...
        for (uint i; i < len;) {
@>          if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
@>              continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
@>              ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

The intention is:
- If `claimedFlows[i]` is true, skip this flow.
- If `claimedSeconds[i] == 0`, treat the full `amounts[i]` as unclaimed and then skip the remaining logic.

However, both `continue` statements appear before the `unchecked { ++i; }` block.

As a result, whenever the code hits either `claimedFlows[i] == true` or `claimedSeconds[i] == 0`, it jumps to the next loop iteration **without incrementing `i`**. The index never changes, so the loop keeps the same `i` value forever.

This causes the for loop to never terminate and eventually revert due to out-of-gas, turning `getUnclaimedAmounts()` into an unusable helper.

### Internal Pre-conditions

There exists at least one flow `i` in a TVS that satisfies one of the following:
- `claimedFlows[i] == true`
- `claimedSeconds[i] == 0`

### External Pre-conditions

_No response_

### Attack Path

1.	The protocol launches a reward or bidding project and distributes TVS NFTs through the normal flow (e.g., `launchRewardProject()`, then `setTVSAllocation()`, then `claimRewardTVS()`).
2.	After distribution, at least one TVS has a flow where either `claimedFlows[i] == true` or `claimedSeconds[i] == 0`. In practice this is easy to hit, as any claimed or certain initialized flows can satisfy this condition.
3.	Anyone calls `getUnclaimedAmounts()` directly, or the owner calls `setAmounts()` or `setUpTheDividends()` on `A26ZDividendDistributor`, both of which rely on `getUnclaimedAmounts()`.
4.	When the loop reaches the problematic flow index, the continue is executed without incrementing `i`, so the loop becomes endless at that index and the transaction reverts with out-of-gas.

### Impact

As soon as there exists a flow in a TVS with `claimedFlows[i] == true` or `claimedSeconds[i] == 0`, `getUnclaimedAmounts()` and all functions that depend on it become unusable.

### PoC

Because `getUnclaimedAmounts()` also suffers from an ABI mismatch issue with `allocationOf()`, that bug must be fixed first to avoid immediate reverts and make this PoC executable.

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
This ensures `getUnclaimedAmounts()` no longer reverts due to ABI mismatch, so we can run this PoC normally and observe the out-of-gas behavior caused by the infinite loop.

Now, to demonstrate the scenario described in the **Attack Path** section, create a test file `PoC.t.sol` in the `test` folder with the following code and run it with:

`forge test --mt test_GetUnclaimedAmountsInfiniteLoop  --force -vv`

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
    uint256 constant PROJECT_ID = 0;

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

    function test_GetUnclaimedAmountsInfiniteLoop() public {
        uint256 KOLS_NUM = 100;
    
        uint256 rewardPerKol = 100e18;
        uint256 totalTVSAllocation = KOLS_NUM * rewardPerKol;
        uint256 vestingPeriod = 30 days;

        // Prepare KOL addresses and allocation amounts
        address[] memory kols = new address[](KOLS_NUM);
        uint256[] memory amounts = new uint256[](KOLS_NUM);

        for (uint i; i < KOLS_NUM; i++) {
            kols[i] = makeAddr(string.concat("kol", vm.toString(i)));
            amounts[i] = 100e18;
        }

        // Project creator launches reward project and sets TVS allocations
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 30 days);

        token.approve(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(PROJECT_ID, totalTVSAllocation, vestingPeriod, kols, amounts);
        vm.stopPrank();

        // Each KOL claims their TVS NFT
        for (uint i; i < KOLS_NUM; i++) {
            vm.startPrank(kols[i]);
            vesting.claimRewardTVS(PROJECT_ID);
            vm.stopPrank();
        }
        
        // For each TVS, there is at least one flow where claimedSeconds[i] == 0
        // and getTotalUnclaimedAmounts() will run out of gas
        distributor.getTotalUnclaimedAmounts();
    }
}
```

Output:
```shell
[FAIL: EvmError: Revert] test_GetUnclaimedAmountsInfiniteLoop() (gas: 1057868528)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 2.76s (1.29s CPU time)

Ran 1 test suite in 2.76s (2.76s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)
```


### Mitigation

For minimal changes, move the index update statement into the for header:
```diff
+		for (uint i; i < len; i++) {
-       for (uint i; i < len;) {
			if (claimedFlows[i]) continue;
			if (claimedSeconds[i] == 0) {
				amount += amounts[i];
				continue;
			}
			uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
			uint256 unclaimedAmount = amounts[i] - claimedAmount;
			amount += unclaimedAmount;
-			unchecked {
-				++i;
-			}
		}
```
Or use `else` branches while keeping the `++i` statement at the bottom.
  