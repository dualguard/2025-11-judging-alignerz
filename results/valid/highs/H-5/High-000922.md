# [000922] [H-4] Incorrect Cumulative Fee Deduction Causes Overcharging in Multi‑Flow TVS
  
  ### Summary

The calculateFeeAndNewAmountForOneTVS function incorrectly subtracts cumulative fees from each flow, causing later flows in multi-flow TVS operations to be overcharged. This breaks the intended per-flow fee logic for splits and merges, leading to users paying more than expected and incorrect fee accounting.

### Root Cause

The function calculateFeeAndNewAmountForOneTVS accumulates the total feeAmount across all flows and then subtracts this running total from each individual flow (newAmounts[i] = amounts[i] - feeAmount). This approach mistakenly applies all previous fees to each subsequent flow instead of calculating and subtracting only the fee for the current flow, resulting in overcharging.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. An attacker initiates a TVS operation involving multiple flows (e.g., splitting a large TVS into several parts).
2. Because the function subtracts the cumulative fee instead of the per‑flow fee, later flows receive increasingly incorrect (much smaller) output amounts.
3. The attacker can deliberately craft multi‑flow operations to amplify these inconsistencies and demonstrate that the protocol is miscalculating fees.
4. This enables malicious actors to cause economic damage, disrupt user balances, or exploit inconsistent fee behavior for arbitrage or griefing, even without direct fund theft.


### Impact

Users are overcharged on TVS operations involving multiple flows, such as splits or merges. The protocol may collect incorrect fees, resulting in potential fund loss for users and misallocation to the treasury. This undermines trust in the protocol and can be exploited by crafting multi-flow TVS operations to amplify fee discrepancies.


### PoC

1. Create a FeesManager.t.sol file inside Test folder .
2.  Run the test using:

```solidity
 forge test --match-test test_IncorrectCumulativeFeeSubtraction --match-path test/FeesManager.t.sol -vvvv
```
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/contracts/vesting/feesManager/FeesManager.sol";

contract FeesManager_PoC is Test {
    
    function test_IncorrectCumulativeFeeSubtraction() public pure {
        uint256 feeRate = 1000; 
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100 ether;
        amounts[1] = 200 ether;
        amounts[2] = 300 ether;
        uint256 length = 3;

        uint256 feeAmount;
        uint256[] memory newAmounts = new uint256[](length);
        
        for (uint256 i; i < length;) {
            uint256 flowFee = amounts[i] * feeRate / 10000;
            feeAmount += flowFee;
            newAmounts[i] = amounts[i] - feeAmount; 
            unchecked { ++i; }
        }

        uint256 expectedFeeAmount;
        uint256[] memory expectedAmounts = new uint256[](3);
        
        for (uint256 i = 0; i < 3; i++) {
            uint256 flowFee = amounts[i] * feeRate / 10000;
            expectedFeeAmount += flowFee;
            expectedAmounts[i] = amounts[i] - flowFee; 
        }

        assert(newAmounts[1] != expectedAmounts[1]);
        assert(newAmounts[2] != expectedAmounts[2]); 
    }
}

```

### Mitigation

The calculation should be updated to compute and subtract fees individually for each flow rather than using a running cumulative total. Specifically, for each flow in the TVS, calculate its fee using the defined feeRate and subtract only that flow’s fee from its amount, while optionally maintaining a separate variable to track the total accumulated fee. This ensures that users are charged correctly for each flow, prevents overcharging on multi-flow TVSs, and guarantees accurate treasury accounting. Implementing this change removes the discrepancy between expected and actual fee deductions and preserves the integrity of TVS operations.

  