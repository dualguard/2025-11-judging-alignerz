# [000657] HIGH Cumulative fee subtraction in `calculateFeeAndNewAmountForOneTVS` corrupts per-flow amounts and can underflow later flows.
  
  ### Summary

Using the cumulative fee accumulator instead of a per-flow fee in `calculateFeeAndNewAmountForOneTVS` will cause incorrect vesting amounts for TVS holders as the contract will repeatedly subtract all prior fees from each flow, shrinking or underflowing later flows and misallocating TVS balances during merge/split operations.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174C6

In `FeesManager.calculateFeeAndNewAmountForOneTVS`, the function aggregates fees for all flows in `feeAmount`, but then subtracts this **cumulative** value from each individual `amounts[i]` instead of subtracting only the fee for that specific flow.

The function as written is:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount; // @audit bug
        }
    }
```

Here:

- On iteration `i`, `feeAmount` already includes fees from indices `< i`.  
- `newAmounts[i] = amounts[i] - feeAmount` subtracts **other flows’ fees** in addition to its own.  

This both miscomputes `newAmounts[i]` and risks underflow for later indices if `feeAmount > amounts[i]`. The correct logic should compute `feeForFlow = calculateFeeAmount(feeRate, amounts[i])`, set `newAmounts[i] = amounts[i] - feeForFlow`, and then accumulate `feeAmount += feeForFlow`.


### Internal Pre-conditions

1. A TVS NFT must have an allocation with at least 2 flows in `allocation.amounts`, so that `length = allocation.amounts.length > 1` and cumulative fees across flows can exceed the later per-flow amounts.

2. The protocol must call a feature that uses `calculateFeeAndNewAmountForOneTVS`, such as `splitTVS` or `mergeTVS`, which passes `feeRate`, `allocation.amounts`, and `allocation.amounts.length` into this helper and then updates `allocation.amounts = newAmounts` and transfers `feeAmount` to the treasury.

3. The fee rate must be non-zero so that `calculateFeeAmount(feeRate, amounts[i]) > 0` for at least one index, making `feeAmount` strictly increasing across iterations.


### External Pre-conditions

none

### Attack Path

1. A user has a TVS NFT with an allocation containing multiple flows, for example `amounts = [A0, A1, A2]`, all positive, and wishes to split or merge the TVS.

2. The user calls `splitTVS` or `mergeTVS` (depending on where `calculateFeeAndNewAmountForOneTVS` is used), passing through the project ID and relevant NFT IDs; the contract computes `uint256[] memory amounts = allocation.amounts;` and `uint256 nbOfFlows = allocation.amounts.length;`.

3. The contract calls `calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows)`, expecting to receive:  
   - `feeAmount` = sum of per-flow fees, and  
   - `newAmounts[i]` = `amounts[i] - feeForFlow(i)` for each index.

4. Instead, on each iteration `i`, `feeAmount` accumulates all previous fees and then the code sets `newAmounts[i] = amounts[i] - feeAmount`.

   - For `i = 0`: `feeAmount = f0`, `newAmounts[0] = A0 - f0` (correct for the first element).  
   - For `i = 1`: `feeAmount = f0 + f1`, `newAmounts[1] = A1 - (f0 + f1)` (subtracts the fee of index 0 as well).  
   - For `i = 2`: `feeAmount = f0 + f1 + f2`, `newAmounts[2] = A2 - (f0 + f1 + f2)`, and so on.  

5. If `A1 < f0 + f1` or `A2 < f0 + f1 + f2`, the subtraction underflows and reverts the entire operation; if not, the new amounts become significantly smaller than intended, distributing an effectively higher effective fee than `feeRate` across later flows.

6. The contract then sets `allocation.amounts = newAmounts` and transfers `feeAmount = f0 + f1 + ...` to the treasury; users end up with less TVS in later flows than the configured fee rate implies, and in some cases the transaction will revert unexpectedly.


### Impact

The TVS holder suffers from incorrect or failing fee application: per-flow vesting amounts are reduced by more than the configured fee rate (or cause underflow and revert) because each flow is charged the cumulative fee of all preceding flows, not just its own fee. This leads to over-charging effective fees on later flows, potential denial of service for split/merge operations due to underflows, and a mismatch between advertised and actual economics of the protocol’s fee mechanism.

### PoC

_No response_

### Mitigation

_No response_
  