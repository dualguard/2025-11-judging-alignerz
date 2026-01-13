# [000961] Flawed logic in `FeesManager::calculateFeeAndNewAmountForOneTVS()` will result DoS
  
  ### Summary

The [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) has a logic flaw in returning an array called `newAmounts`. When user will call the [AlignerzVesting::splitTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) the call flow goes to `FeesManager::calculateFeeAndNewAmountForOneTVS()`. In this function the tx reverts with `FAIL: panic: array out-of-bounds access (0x32)]` error. 

### Root Cause

The root cause of the error is how the (to be returned) array, named `newAmounts` , was initialized. 
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
 @>         newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
In above function we can see that `amounts[i] - feeAmount` was directly stored in `newAmounts[i]`, but note that this `newAmounts[]` was not initialized with any length. Means the value of `amounts[i] - feeAmount`  trying to access such place which is not exist yet in `newAmounts[]`. 

### Internal Pre-conditions

There are no internal pre-conditions. 

### External Pre-conditions

There are no internal external pre-conditions.

### Attack Path

There is no attack path, it will be a simple revert. 

### Impact

All tx to `FeesManager::calculateFeeAndNewAmountForOneTVS()` will always revert. It means `mergeTVS` and `splitTVS` operations will always revert. 

### PoC

Create a test file in `./test` directory and paste this code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./AlignerzVestingProtocolTest.t.sol";

contract RewardProjectTest is AlignerzVestingProtocolTest {
        function test_splitTVS() public {
        uint currentTime = block.timestamp;
        vm.prank(projectCreator);
        // Launching reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            currentTime,
            30 days  // claimWindow
        );

        // Setting TVS allocation
        address[] memory kolTVS = new address[](1);
        uint[] memory TVSamounts = new uint[](1);

        kolTVS[0] = makeAddr("KOL-1");
        
        TVSamounts[0] = 1000e18;
        
        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, 1000e18, 60 days, kolTVS, TVSamounts);

        vm.warp(currentTime + 25 days);

        vm.prank(makeAddr("KOL-1"));
        vesting.claimRewardTVS(0);

        assertEq(nft.extOwnerOf(1), makeAddr("KOL-1"));

        uint[] memory percentages = new uint[](5);
        percentages[0] = 2000;
        percentages[1] = 2000;
        percentages[2] = 2000;
        percentages[3] = 2000;
        percentages[4] = 2000;
        assertEq(vesting.splitFeeRate(),0,"Split fee error");
        vm.prank(makeAddr("KOL-1"));
        vesting.splitTVS(0,percentages,1);
    }

}
```
Run the test:
```solidity
forge clean && forge build
forge test --mt test_splitTVS -vvvv
```
Output:
```solidity
├─ [14974] ERC1967Proxy::fallback(0, [2000, 2000, 2000, 2000, 2000], 1)
    │   ├─ [14194] AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Revert] panic: array out-of-bounds access (0x32)
```

### Mitigation

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       newAmounts = new uint[](amounts.length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  