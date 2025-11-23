# [000160] Last NFT owner is unable to receive dividends
  
  ### Summary

The NFT counting is off by one, causing the issue

### Root Cause

Looking at the setDividends function:
```solidity
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```

The loop iterates over every single NFT mint in order to calculate dividends. But based on the ERC721A implementation, the NFT of ID 0 does not exist, and rather it begins at 1. As a consequence, the loop never iteracts over the last valid ID, which is `len` and not `len - 1`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. 2 users mint their NFTs for dividends, their IDs being 1 and 2
2. When the function runs, the setup function will ask the NFT contract for the total minted, which is 2
3. Now the loop enters, it will iterate through ID = 0, which is never valid, and ID = 1, which it is
4. Loop ends without awarding ID 2

### Impact

Last ID owner will never receive funds

### PoC

_No response_

### Mitigation

Let the loop go through one last iteration
```solidity
for (uint i; i <= len;) {
```
  