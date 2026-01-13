# [000423] Infinite loop vulnerability in `_ownershipOf` function
  
  ### Summary

The `ERC721A._ownershipOf` function contains a `while(true)` loop without proper bounds checking that can lead to infinite iteration, causing transaction reversion due to out-of-gas errors and effectively making certain tokens permanently inaccessible

### Root Cause

The vulnerability stems from the backward lookup mechanism in the `_ownershipOf` function.
The loop assumes that there will always be an ownership record with a non-zero address before reaching `_startTokenId()`, but this assumption can be violated in certain edge cases, particularly:

- When tokens are burned in specific sequences
- When ownership slots are not properly initialized
- When integer underflow occurs in the unchecked block


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

An adversarial KOL could transfer the minted NFTs elsewhere, and after the `RewardProject` ends, when the owner attempts to distribute tokens and performs the `burn` process, the attacker could exploit the ownership query for those already-moved tokenIds to carry out a gas-griefing attack.

### Impact

Denial of Service:
- Specific tokens become permanently inaccessible
- All operations on affected tokens (transfer, burn, approve) fail

Gas Griefing:
- Transactions attempting to query affected tokens consume all available gas
- Cost of failed transactions falls on users
- Can be weaponized to drain user funds through gas costs

Protocol Failure:
- Core functionality (ownerOf, balanceOf, transfers) becomes unusable

### PoC

_No response_

### Mitigation

Add loop bounds or iteration limit. Also, using explicit ownership storage is the best practice.
  