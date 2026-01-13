# [000620] `allocationOf[nftId]` not updated in mergeTVS function
  
  ### Summary

The mapping `allocationOf[nftId]` is not updated on mergeTVS, while it's updated in splitTVS
The new allocation parameter pushed will not be shown on the `allocationOf` mapping

### Root Cause

mapping `allocationOf` not updated in mergeTVS

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

someone calls allocationOf[nftId] to get the allcation of the nfts, but it's not up to date when the nft is merged

### Impact

allocationOf[nftId] not up to date

### PoC

_No response_

### Mitigation

update allocationOf[nftId] in mergeTVS
  