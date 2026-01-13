# [000812] Dividend distribution can be permanently DoSed
  
  ### Summary

`splitTVS()` allows users to split an NFT using percentages.
Currently, there is no check that ensures the resulting split amounts are non-zero.

An attacker can split his NFT into many zero-amount NFTs. When the owner sets amounts to distribute in `A26ZDividentDistributor.sol`
```solidity
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            // q oog ako ostavqt mnogo mnogo nftta?
            unchecked {
                ++i;
            }
        }
```
It will loop through all minted NFTs, including the attacker’s enormous zero-amount NFTs. The gas limit on Arbitrum is 32 million, therefore an attacker can easily permanently block dividend distribution.


### Root Cause

The owner has to loop through every single NFT, including the attacker’s zero-amount NFTs.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

In Summary

### Impact

Dividend distribution can be permanently DoSed.

### PoC

_No response_

### Mitigation

_No response_
  