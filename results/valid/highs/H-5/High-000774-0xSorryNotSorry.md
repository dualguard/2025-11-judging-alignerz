# [000774] Cumulative Fee subtraction overcharges user vesting flows
  
  ### Summary

The cumulative use of `feeAmount` inside the fee helper will cause an excessive loss of tokens for TVS holders during merge and split operations as each later flow is charged not only its own fee but also all previous flows’ fees, so users lose more than the configured fee rate while the protocol only accounts for the normal total fee.

### Root Cause

In `FeesManager.sol:L169-L173` the helper `calculateFeeAndNewAmountForOneTVS` accumulates `feeAmount` over all flows and then subtracts the full cumulative value from each `amounts[i]`, instead of subtracting only the per-flow fee for that index.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
>>      newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

Once the array-initialization and iterator bugs are fixed so that this loop runs as intended, the returned `feeAmount` is equal to the sum of the per-flow fees (correct), but the actual reduction applied to user amounts is larger than this total because flow `i` is reduced by `f0 + f1 + ... + fi` instead of just `fi`. This behavior affects both `mergeTVS` and `splitTVS` in `AlignerzVesting.sol` because they rely on `newAmounts` as the updated vesting amounts and transfer only the total `feeAmount` to the treasury.


### Internal Pre-conditions

1. A TVS allocation exists with at least two flows (`amounts.length >= 2`) and non-zero amounts so that there are “later” flows which can be overcharged by the cumulative fee.


### External Pre-conditions

1. The admin configures a non-zero `splitFeeRate` or `mergeFeeRate` in basis points and users actually call `splitTVS` or `mergeTVS` on multiflow TVSs.

### Attack Path

1. The owner deploys or upgrades the contract to a version where `calculateFeeAndNewAmountForOneTVS` now allocates `newAmounts` and increments `i` but still uses `newAmounts[i] = amounts[i] - feeAmount;` with `feeAmount` cumulative.  
2. A user owns an NFT whose TVS allocation has multiple flows with amounts `[a0, a1, a2]` and calls `splitTVS` or `mergeTVS` with a non-zero fee rate (for example 10%).  
3. Inside the helper, the contract computes:
   - flow 0: `fee0 = a0 * rate / 10_000`, `feeAmount = fee0`, `newAmounts[0] = a0 - fee0`  
   - flow 1: `fee1 = a1 * rate / 10_000`, `feeAmount = fee0 + fee1`, `newAmounts[1] = a1 - (fee0 + fee1)`  
   - flow 2: `fee2 = a2 * rate / 10_000`, `feeAmount = fee0 + fee1 + fee2`, `newAmounts[2] = a2 - (fee0 + fee1 + fee2)`  
4. The returned `feeAmount` equals `fee0 + fee1 + fee2`, but the user’s new total allocation is `Σ newAmounts[i] = Σ ai - (3*fee0 + 2*fee1 + 1*fee2)`, which is strictly less than `Σ ai - (fee0 + fee1 + fee2)`, so the user loses extra tokens beyond the configured fee.  
5. Only the configured `feeAmount` is transferred to the treasury; the “extra” tokens remain unaccounted for in users’ allocations and can end up as hidden protocol surplus when profits are later withdrawn.


### Impact

TVS holders suffer a systematic loss larger than the advertised fee rate whenever they merge or split multi-flow TVSs, as each later flow is charged multiple times; the extra lost tokens remain in the contract and can eventually be taken by the protocol as profit, meaning users lose value without a corresponding gain described by the fee configuration (severity: Medium, overcharging but no external attacker profit).

### PoC

This POC  reproduces the buggy cumulative logic in isolation (with the array and iterator already fixed so I needed to use the fixed version of the function  named as `_buggyCalculateFeeAndNewAmountForOneTVS`) and shows that the user’s total loss is strictly higher than the configured total fee.
Please add below inside `test` folder as `FeesManagerBug.t.sol` and run with `forge test -vvvv --match-path test/FeesManagerBug.t.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract FeesManagerBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public kol;

    uint256 constant REWARD_PROJECT_ID = 0;

    function setUp() public {
        owner = address(this);
        kol = makeAddr("kol");

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        token.approve(address(vesting), type(uint256).max);
    }


    function _buggyCalculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) internal pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            uint256 fee = amounts[i] * feeRate / 10_000;
            feeAmount += fee;
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                ++i;
            }
        }
    }

    function test_cumulative_fee_overcharges_later_flows() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100 ether;
        amounts[1] = 100 ether;
        amounts[2] = 100 ether;

        uint256 feeRate = 1000; // 10% in basis points

        (uint256 feeAmount, uint256[] memory newAmounts) =
            _buggyCalculateFeeAndNewAmountForOneTVS(feeRate, amounts, amounts.length);

        uint256 originalTotal;
        uint256 newTotal;
        for (uint256 i; i < amounts.length; ++i) {
            originalTotal += amounts[i];
            newTotal += newAmounts[i];
        }

        // Total configured fee (sum of per-flow fees)
        uint256 expectedFee = 0;
        for (uint256 i; i < amounts.length; ++i) {
            expectedFee += (amounts[i] * feeRate / 10_000);
        }
        assertEq(feeAmount, expectedFee, "Returned feeAmount must equal sum of per-flow fees");

        // Check that the user's loss is larger than the configured fee.
        uint256 userLoss = originalTotal - newTotal;
        assertGt(userLoss, feeAmount, "User loses more than the total configured fee");
    }
}
```

The result:
```bash
Ran 1 test for test/FeesManagerBug.t.sol:FeesManagerBugTest
[PASS] test_cumulative_fee_overcharges_later_flows() (gas: 20503)
Traces:
  [20503] FeesManagerBugTest::test_cumulative_fee_overcharges_later_flows()
    ├─ [0] VM::assertEq(30000000000000000000 [3e19], 30000000000000000000 [3e19], "Returned feeAmount must equal sum of per-flow fees") [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assertGt(60000000000000000000 [6e19], 30000000000000000000 [3e19], "User loses more than the total configured fee") [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.30s (88.42µs CPU time)

Ran 1 test suite in 2.30s (2.30s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

<img width="1201" height="197" alt="Image" src="https://github.com/user-attachments/assets/bba9ac64-c2a9-48cd-bd57-5242e3c82a21" />

### Mitigation

Rewrite the function as;
```diff
-    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
-        for (uint256 i; i < length;) {
-            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-            newAmounts[i] = amounts[i] - feeAmount;
-        }
-    }
+    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+        newAmounts = new uint256[](length);
+        for (uint256 i; i < length;) {
+            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
+            feeAmount += fee;
+            newAmounts[i] = amounts[i] - fee;
+            unchecked {
+                ++i;
+            }
+        }
+    }
```
  