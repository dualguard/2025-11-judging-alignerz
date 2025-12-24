# [000446] Loss of fees income for the protocol
  
  ### Summary

During splitting and merging, the fees calculated are rounded down. This will result in a loss of funds for the protocol. The smaller users merge their funds, the less fees they will pay.

### Root Cause

Fees are rounded down.

### Attack Path

Alice may want to frequently transfer allocations to various people. Since she realized the protocol rounds down the fees, and has a maximum of 2% fee for merging/splitting, she directly split her NFTs into values small enough so she does not have to pay fees after (she only pays for the first time it's done). 

Then, when she has to send X amount of allocations, she can either split it into smaller parts if needed, or merge them before sending it and she **won't pay any more fees** in both cases. 

### Impact

Reduced fees for the protocol

### Mitigation

Round up when calculating fees for merging and splitting NFTs.

  