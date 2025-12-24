# [000634] Users will lose exponentially increasing funds during merge/split operations due to compounding fee calculation erro
  
  ### Summary

The compounding fee accumulation bug in `calculateFeeAndNewAmountForOneTVS()`  will cause exponentially increasing fund loss for users as the function accumulates fees across all flows but deducts the cumulative total from each individual flow, resulting in users paying `fee × N(N-1)/2`  extra tokens where N is the number of flows, instead of the intended `fee × N` total.

### Root Cause


In [FeesManager.sol:169-172](https://github.com/dualguard/2025-11-alignerz-olaoyesalem/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-#L172) the feeAmount variable accumulates across all iterations `(feeAmount += calculateFeeAmount(...))`, but this cumulative total is then subtracted from each individual flow `(newAmounts[i] = amounts[i] - feeAmount)`, causing:

- **Flow 0** : Loses `fee₀` (correct)
- **Flow 1** : Loses `fee₀ + fee₁` (should only lose fee₁)
- **Flow 2** : Loses `fee₀ + fee₁ + fee₂` (should only lose fee₂)
- **Flow N**: Loses sum of all previous fees (exponential loss)

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // ← Accumulating total
        newAmounts[i] = amounts[i] - feeAmount;                // ← Subtracting CUMULATIVE fee instead of individual fee!
    }
}
```

Example with 3 flows of 1000 tokens each, 1% fee (10 tokens per flow):


Iteration | Fee Calculation | Cumulative Fee | Deduction | Expected Result | Actual Result | Loss
-- | -- | -- | -- | -- | -- | --
Flow 0 | 10 | 10 | 1000 - 10 | 990 | 990 | 0 ✓
Flow 1 | 10 | 20 | 1000 - 20 | 990 | 980 | 10 ✗
Flow 2 | 10 | 30 | 1000 - 30 | 990 | 970 | 20 ✗


Total: User gets 2940 instead of 2970 → 30 extra tokens lost (100% overcharge)


### Internal Pre-conditions

- User needs to own multiple NFTs with vesting flows to trigger merge or split operations
- Merge fee rate (mergeFeeRate) or split fee rate (splitFeeRate) needs to be set to greater than 0 by owner (typical: 1-2% or 100-200 basis points)
- NFTs need to have at least 2 vesting flows to demonstrate the compounding effect (more flows = exponentially worse)

### External Pre-conditions

None

### Attack Path

- User participates in multiple projects and receives separate `NFT` vesting positions (e.g., 3 NFTs from different bidding pools).

- User calls `mergeTVS()` to consolidate vesting positions into a single `NFT` for gas efficiency.
- Protocol owner has set `mergeFeeRate `to 100 basis points (`1%`) as reasonable protocol fee.
- Contract calls `calculateFeeAndNewAmountForOneTVS()` with the user's flow amounts `[1000, 1000, 1000] `and fee rate `100`.
- 
Function calculates fees correctly but applies them incorrectly:
`newAmounts[0] = 1000 - 10 = 990 ✓`
`newAmounts[1] = 1000 - 20 = 980 ✗ (loses double)`
`newAmounts[2] = 1000 - 30 = 970 ✗ (loses triple)`

- User's merged NFT receives only `2940` tokens instead of expected `2970` tokens
- 30 tokens vanish - not collected as fees (protocol expects 30 total), not given to user
- Same bug applies to `splitTVS()` operations
Scale with More Flows:

```
5 flows: Expected fee 50, actual loss 100 (100% overcharge)
10 flows: Expected fee 100, actual loss 450 (450% overcharge)
20 flows: Expected fee 200, actual loss 1900 (950% overcharge)
```

### Impact


- Users with `N` flows lose `fee × N(N-1)/2` extra tokens beyond the intended fee:

Concrete Examples with 1% Fee:

Flows (N) | Amount per Flow | Expected Fee | Actual Loss | Extra Loss | Overcharge %
-- | -- | -- | -- | -- | --
2 | 1000 | 20 | 30 | 10 | 50%
3 | 1000 | 30 | 60 | 30 | 100%
5 | 1000 | 50 | 150 | 100 | 200%
10 | 1000 | 100 | 550 | 450 | 450%
20 | 1000 | 200 | 2000 | 1800 | 900%



- Users lose significantly more funds than intended protocol fee
- Loss scales quadratically with number of flows
- Affects both merge and split operations

### PoC

Due to the fact that there is a bug which I have submitted in the other issue )in `mergeTVS()` which won't allow  to run the coded PoC

```solidity

Fee per flow = F
Number of flows = N
Expected total fee = F × N

Actual deductions:
Flow 0: F
Flow 1: 2F  
Flow 2: 3F
...
Flow (N-1): N×F

Actual total loss = F + 2F + 3F + ... + N×F = F × (1+2+3+...+N) = F × N(N+1)/2

Extra loss = F × N(N+1)/2 - F × N = F × N × (N+1-2)/2 = F × N(N-1)/2
```

### Mitigation

Fix the compounding bug in `calculateFeeAndNewAmountForOneTVS`:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // Initialize the return array
    newAmounts = new uint256[](length);
    
    for (uint256 i; i < length;) {
        // Calculate fee for THIS flow only (not cumulative)
        uint256 currentFee = calculateFeeAmount(feeRate, amounts[i]);
        
        // Accumulate total fees for return value
        feeAmount += currentFee;
        
        // Deduct ONLY this flow's fee (not cumulative)
        newAmounts[i] = amounts[i] - currentFee;
        
        unchecked {
            ++i;
        }
    }
}
```
  