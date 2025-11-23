# [000132] Gas-intensive iteration in `getTotalUnclaimedAmounts()` risks dividend setup failure
  
  ### Summary

`A26ZDividendDistributor.getTotalUnclaimedAmounts()` performs a linear scan over all minted NFTs with multiple external calls per token and no batching. Measured gas usage shows a fixed base cost plus a significant per-NFT increment, which can cause the call to approach or exceed the block gas limit as the number of NFTs grows. This behavior is assessed as **Medium** severity because it may impair dividend configuration while not directly enabling fund theft or privilege escalation.


### Root Cause

In `getTotalUnclaimedAmounts()`, the contract calls `nft.getTotalMinted()` and then loops from `0` to `len - 1`, calling `safeOwnerOf(i)` and, when the NFT is owned, `getUnclaimedAmounts(i)`, which in turn issues six `vesting.allocationOf(i)` calls and processes allocation arrays. In a concrete Foundry-based measurement with a minimal but representative mock setup, the gas usage was:

- `getTotalUnclaimedAmounts()` with **0 NFTs minted**: **12,833 gas** (logged `gas_used_getTotalUnclaimedAmounts_0NFT`)
- `getTotalUnclaimedAmounts()` with **10 owned NFTs**, each with a single active vesting flow: **793,402 gas** (logged `gas_used_getTotalUnclaimedAmounts_10NFT`)

The additional gas attributable to 10 NFTs is exactly `793,402 − 12,833 = 780,569` gas, i.e. **78,056.9 gas per owned NFT iteration** in this scenario. On a 30,000,000 gas block limit, this linear scaling implies a practical ceiling of approximately  
\((30,000,000 − 12,833) / 78,056.9 ≈ 384\) owned NFTs before a full traversal risks running out of gas.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. The protocol mints NFTs over time as users participate, and each NFT accumulates vesting flows.
2. The number of owned NFTs and their allocation complexity grow, increasing the gas cost of each full traversal in `getTotalUnclaimedAmounts()`.
3. An operator calls a management function (e.g., one that internally relies on `getTotalUnclaimedAmounts()` for dividends).
4. The call attempts to process all NFTs in one transaction; once the aggregate cost exceeds the block gas limit (e.g., around 384 NFTs under the measured conditions), the transaction runs out of gas and reverts, blocking this operational path.

### Impact

Severity is **Medium** because the function’s gas growth DOSs or prevents the successful execution of any flows that depend on `getTotalUnclaimedAmounts()` (for example, dividend setup), especially at higher but still very realistic NFT counts. 

### PoC

_No response_

### Mitigation

Consider refactoring `getTotalUnclaimedAmounts()` to use bounded, batched processing with explicit index ranges.
  