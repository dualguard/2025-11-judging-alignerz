# [001064] Frontrunning finalizeBids() Allows Attackers to Override Legitimate Top Bids
  
  ### Summary

Because `finalizeBids()` is a public mempool transaction, an attacker can frontrun the finalization by placing a new bid with a slightly longer vesting period (next multiple of `vestingPeriodDivisor`), guaranteeing that their bid is prioritized over legitimate users—even at the last second.

This completely defeats the intended fairness of the bidding process and allows last-moment bid manipulation, especially harmful in highly discounted pools.

> Clearing confusion about `finalizeBids(...)` and `updateProjectAllocations(...)`.
In `finalizeBids(...)` Owner inputs an array of empty bytes of a lenght equal to the nb of pools. To close the project and be sure bids stopped completely. `finalizeBids(...)` will then emit a `BiddingClosed` event that will be listened to by the backend to do the calculations for refund amountd and TVS amounts for each user depending on their bid. Then 5minutes later, after the off-chain calculations are done. Owner will call `updateProjectAllocations(...)` passing the merkleroots generated off-chain post-calculations. The backend will listen to `AllPoolAllocationsSet` to allow the users to be able to call `claimRefund(...)` or `claimNFT(...)`. This is desing choice, the updateProjectAllocations will not be called mid claim, only once, right after `finalizeBids(...)` which I agree can be refactored. This is a design choice.

This said by protocol team in discord.

> To prevent last-minute bid manipulation, bidding window will be hashed.      

This is from Alignerz whitepaper. According to whitepaper they are implementing some mechanism to prevent this issue but this can still be exploited by frontrunning. 

### Root Cause

Bidding does not actually end until the owner calls `finalizeBids()`.
There is no on-chain mechanism preventing new bids from being submitted right before that transaction is mined.
Because `finalizeBids()`:
- is publicly visible in the mempool,
- is gas-raceable,
an attacker can insert a new higher-priority bid (higher vesting length or equal amount + longer vesting) immediately before the project is closed.

### Internal Pre-conditions

1. A bidding project is active and at least one pool is open.
2. Owner must call `finalizeBids(projectId)` for that project.

### External Pre-conditions

1. Attacker monitors mempool for the owner's `finalizeBids()` transaction.
2. Attacker is able to send a higher-gas bid transaction.
3. Network permits frontrunning (always true on public L1/L2 EVM chains).

### Attack Path

1. Many users bid normally.
2. Honest users place legitimate bids with their chosen vesting periods.
3. Owner initiates `finalizeBids(projectId)` -> visible in mempool.
4. Attacker sees the transaction and reads the current highest-priority bids (all public).
5. Attacker submits a bid with slightly longer vesting period, (which is a multiple of `vestingPeriodDivisor`)
6. Attacker's bid lands before `finalizeBids()`.
7. `finalizeBids()` executes -> bidding closes.
8. Backend calculates Merkle allocations including the attacker's manipulated bid.
9. Legitimate bidders are kicked out of the winning slots.
10. Attacker receives the discounted allocation. (If pool have discounts)

### Impact

This vulnerability directly enables last-second manipulation of the bidding outcome.
An attacker can:
- consistently override legitimate bidders,
- especially target high-discount pools such as `A26Z Pool 1 (80% discount + full refund),`
- gain outsized allocations at nearly zero risk,
- push honest users out of allocation entirely.
Because discounted pools distribute a fixed quantity, one attacker can steal the entire pool allocation by manipulating the last moment.
This breaks:
- the fairness guarantees expected in an IWO sale,
- the economic design of allocation priority,
- the trust assumptions for retail participants.

### PoC

_No response_

### Mitigation

_No response_
  