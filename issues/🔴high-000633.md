# [000633] Any caller will trigger an out-of-gas denial of service for vesting NFT operations
  
  ### Summary

The missing loop increment in `calculateFeeAndNewAmountForOneTVS` will cause an infinite loop (out-of-gas revert) for vesting NFT holders as any user calling `splitTVS` or `mergeTVS` will enter a non-terminating fee calculation path.

### Root Cause

In the `feeManager.sol`

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    // newAmounts not allocated here, but focus of this report is ONLY the loop increment bug.
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        // ... writes ...
        // MISSING: i++ (or unchecked { ++i; })
    }
}
```

- Loop termination variable i never changes → infinite loop → eventual out-of-gas revert.

### Internal Pre-conditions

- A vesting NFT with at least one flow exists (so length > 0).
- User invokes splitTVS or mergeTVS (both call calculateFeeAndNewAmountForOneTVS).
- Contract retains the buggy loop (no increment added).

### External Pre-conditions

None

### Attack Path

- Victim calls splitTVS(projectId, percentages, nftId) (or initiates a merge).
- Execution enters calculateFeeAndNewAmountForOneTVS.
`for (uint256 i; i < length;)` evaluates with `i = 0 `and `length >= 1.`
- Body executes; on next iteration condition still `0 < length `(unchanged).
- Repeats until transaction gas is exhausted → revert.
=  Operation (split/merge) is unusable (DoS). Repeated attempts waste user gas.

### Impact

- The affected party (vesting NFT holder) cannot reorganize vesting flows (split/merge) and loses the entire transaction gas each attempt. Protocol functionality degradation: dynamic composability disabled. Economic loss equals wasted gas per attempt; strategic operations (consolidation, redistribution) blocked.

### PoC

_No response_

### Mitigation

Add a loop increment; prefer explicit increment with optional unchecked:

```solidity
for (uint256 i; i < length; ) {
    uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
    feeAmount += fee;
    newAmounts[i] = amounts[i] - fee;
    unchecked { ++i; }
}
```
  