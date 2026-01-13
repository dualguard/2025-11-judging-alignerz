# [000139] Multiple lines of "dead code" exist
  
  ### Summary

Multiple functions in the protocol can never be triggered or don't serve any purpose in the current implementation:

The following functions can never be called as they are internal, but are never called in the contract:
`ERC721A::_numberMinted`
`ERC721A::_numberBurned`
`ERC721A::_getAux`
`ERC721A::_setAux`

The `AlignerzVesting::withdrawStuckETH` function, as well as the `receive` function, are not needed as the protocol doesn't use native tokens in any way 

### Root Cause

.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

There is no direct security impact, but the contract is inflated through the dead code, increasing deployment cost and opening up possible attack surfaces (Native token sweeping function)

### PoC

_No response_

### Mitigation

Remove the mentioned functions
  