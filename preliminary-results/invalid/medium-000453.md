# [000453] Users can still claim dividends if they burn their NFT before the end of the vesting period
  
  ### Summary

NFT holders are entitled to earn dividends proportionally to the amount linked to their NFTs. However, while they have dividends pending to be claimed, they may burn their NFT by either merging it or claiming its underlying tokens, but they will still be entitled to earn the dividends even though the NFT does not exist anymore.

### Root Cause

`claimDividends` sends tokens to user who does not own the NFT.

### Attack Path

1. Alice mints NFT
2. Dividends are send to the contract and pending to be claimed by users
3. Alice burns her NFT
4. Alice still earns her dividends based on the vesting schedule

### Impact

User earns rewards that are not intended to them anymore

### Mitigation

Checks the dividends' NFT are not burned before claiming it.
  