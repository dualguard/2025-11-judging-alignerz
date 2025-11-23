# [000520] Missing Allocation Validation in claimNFT Enables Unlimited Token Minting
  
  ### Summary

`claimNFT` mints NFTs for any `amount` in the Merkle proof without verifying it matches the user's bid or pool allocation, allowing arbitrary token vesting.


### Root Cause

No cross-check between proof `amount` and on-chain `bid.amount` or pool totals.


### Internal Pre-conditions

- Project closed with allocation roots.

### External Pre-conditions

- User bid any amount.
- Tree includes oversized `amount`.

### Attack Path

User claims NFT with huge `amount` via proof, then claims tokens over time.


### Impact


Infinite minting of project tokens, diluting supply and stealing from treasury.

### PoC

No response

### Mitigation

Validate in `claimNFT`:
```diff
+ require(amount == calculateAllocation(bid.amount, poolPrice), "Mismatch");
```
  