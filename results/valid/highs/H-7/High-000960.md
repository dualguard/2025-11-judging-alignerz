# [000960] Infinite loop will result in out of gas error i.e DoS
  
  ### Summary

The `for` loop in [FeesManager::calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) using a  `for` loop which iterates to the infinity results in out of gas error. 

### Root Cause

```solidity
for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
}
```
Here we can see that the `i` never increments, as the value of `i` is 0 at the time of declaration and never increments so it satisfies `i < length` always. Means it results in an infinite loop. 

### Internal Pre-conditions

Not needed. 

### External Pre-conditions

Smart contracts are live. 

### Attack Path

Not needed. 

### Impact

Tx for `mergeTVS` & `splitTVS` will always revert. 

### PoC

To run the POC correctly we need to refine the `calculateFeeAndNewAmountForOneTVS` as this function already has an issue, if we don't write the fix in the code the PoC will not work. I already submitted that issue separately in a report(look for this title: Flawed logic in FeesManager::calculateFeeAndNewAmountForOneTVS() will result DoS). So here I just write the diff. Update the function to this:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint[](amounts.length);  // Add this line
        for (uint256 i; i < length; ) {   // @audit-issue
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
After that create a test file in ./test directory and paste this code:
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
Run this test:
```solidity
forge clean && forge build
forge test --mt test_splitTVS -vvvv
```
Output:
```solidity
├─ [1039592297] ERC1967Proxy::fallback(0, [2000, 2000, 2000, 2000, 2000], 1)
    │   ├─ [1039591516] AlignerzVesting::splitTVS(0, [2000, 2000, 2000, 2000, 2000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```

### Mitigation

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
-       for (uint256 i; i < length;) {
+       for (uint256 i; i < length; i++) { 
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  