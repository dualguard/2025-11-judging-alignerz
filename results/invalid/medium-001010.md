# [001010] Vested tokens can be claimed right before the a TVS is sold, resulting in the buyer receiving a TVS with reduced value.
  
  ### Summary

The value of a TVS is given by the amount of tokens allocated to it but not claimed yet. This includes both vested but not claimed as well as unvested tokens. 

The `claimTokens()`  may be executed right before the TVS is sold, thus reducing the TVS value. The buyer gets less value for what he paid for. 


### Root Cause

A TVS can be listed on a NFT marketplace for a long period of time until it's bought. In this time the TVS unlocks tokens which can be claimed anytime. 
A [claimTokens()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) tx may be executed right before the TVS is bought, causing the buyer to receive only a fraction of the value paid.

The example provided in the Attack Path below provides more context about how this vulnerability may happend. 

### Internal Pre-conditions

None. 

### External Pre-conditions

1. TVS must be listed for sale.

### Attack Path

1. Alice hold TVS1 with unvested tokens worth 1000 USD. For simplicity let's consider vesting period is 1000 seconds. 
2. Alice list the TVS1 on a marketplace for 900 USD
3. 400 seconds has passed => claimable tokens are worth 400 USD.
4. Bob buys buys TVS1, paying 900USD hoping to receive a token worth 1000 USD claimable at the end of vesting period. 
5. Right before Bob's tx is received and processed by the L2 sequencer, Alice's `claimTokens()` tx is executed. After the block is minted:
- Alice claimed tokens worth 400 USD;
- Bob paid 900 USD for a TVS which, after vesting period, will unlock tokens worth `1000 - 400 = 600` USD

Bob lost 600 - 900 = -300 USD, without considering he has to wait for tokens to be fully vested. 

### Impact

TVS buyers risk obtaining significantly less value than the amount they paid 

### PoC

_No response_

### Mitigation

Currently I don't see how the vulnerability described above can be mitigated. 
The `AlignerzVesting` contract can't be informed when a TVS is listed for sale to deny the claiming of tokens (and it shouldn't deny it imo!). 
The protocol should ensure that buyers have access to all relevant information to make an informed decision. Ideally, the full TVS details should be embedded in the token URI/metadata. A good example is the [Sablier](https://opensea.io/item/ethereum/0xcf8ce57fa442ba50acbc57147a62ad03873ffa73/840) Vesting Streams NFTs.
  