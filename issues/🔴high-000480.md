# [000480] Incorrect NFT ID Range Iteration Skips the Final Token in Dividend Distribution
  
  ### Summary

The `A26ZDividendDistributor` contract contains an off by one error in its loops that iterate over NFT IDs. It assumes NFT IDs start at 0, but the underlying ERC721A implementation starts minting at ID 1. Consequently, the loop terminates one step early, systematically excluding the last minted NFT from receiving any dividend allocations.

### Root Cause

The loops in _setDividends and getTotalUnclaimedAmounts iterate from i = 0 to i < nft.getTotalMinted().

ERC721A (used by AlignerzNFT) overrides _startTokenId() to return 1.

If 10 NFTs are minted, the IDs are 1, 2, ..., 10.

The loop runs for i = 0, 1, ..., 9.

i = 0 is queried (invalid/unowned).

i = 10 is never queried.

### Internal Pre-conditions

The AlignerzNFT contract must be deployed with ERC721A logic where _startTokenId() is 1.

### External Pre-conditions

t least one NFT must be minted.

The owner calls setUpTheDividends, setAmounts, or setDividends.

### Attack Path

NA

### Impact

The holder of the most recently minted NFT (highest ID) is permanently excluded from all dividend distributions.

This leads to a loss of funds for that user and incorrect accounting of total unclaimed amounts.

### PoC

NA

### Mitigation

Update the loop logic to correctly align with the NFT ID range.

```solidity
uint256 totalMinted = nft.getTotalMinted();
for (uint i = 1; i <= totalMinted;) { // Start at 1, include totalMinted
    // ... logic ...
    unchecked { ++i; }
}

```
  