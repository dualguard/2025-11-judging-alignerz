# [000981] Token holders will bypass pause mechanism in AlignerzNFT [Low]
  
  ### Summary

The missing pause enforcement in AlignerzNFT transfer functions will cause a failure of operational control for administrators as token holders will transfer tokens even when the contract is paused.

### Root cause

In `protocol/src/contracts/nft/AlignerzNFT.sol#L80-L88` the pause mechanism documentation claims to halt "minting and transfers" but the implementation only enforces the paused state on mint and burn operations through the `whenNotPaused` modifier, while ERC721A transfer functions lack any pause validation.

### Internal pre-conditions

1. Owner needs to call `pause()` to set the contract to be in paused state
2. Token holders need to own NFTs to have tokens available for transfer

### External pre-conditions

None required.

### Attack path

1. Owner calls `pause()` to halt contract operations including transfers
2. Token holder calls `transferFrom()` or `safeTransferFrom()` with valid token ownership and the transfer succeeds despite paused state

### Impact

The administrators suffer an inability to control token transfers during emergency situations. The pause mechanism fails to provide complete operational control when administrators expect to halt all token movement, undermining emergency stop functionality and preventing effective incident response measures.

### Mitigation

Consider implementing pause enforcement on token transfers to align with the documented behavior. One approach could be to override the `_beforeTokenTransfers()` function from ERC721A to include a pause check:

```solidity
function _beforeTokenTransfers(
    address from,
    address to,
    uint256 startTokenId,
    uint256 quantity
) internal override {
    require(!paused(), "Token transfers paused");
    super._beforeTokenTransfers(from, to, startTokenId, quantity);
}
```

  