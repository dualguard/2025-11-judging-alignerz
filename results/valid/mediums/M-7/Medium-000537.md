# [000537] Calling `finalizeBids()` With an Empty Allocation Array Creates a Bidding Manipulation Window on Polygon chain
  
  ### Summary

Calling `finalizeBids()` with an empty array before off-chain calculations are finalized will cause **last-second bid manipulation** for **all bidders** as **attackers will monitor the mempool, detect the finalize transaction, and quickly submit new bids before it confirms**. [Readme](https://github.com/dualguard/2025-11-alignerz?tab=readme-ov-file#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed) saysPolygon is one of the chains it will be launched on and it has a mempool just like Eth chain.

### Root Cause

The protocol triggers the bid closing by calling [`finalizeBids()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783) **before** backend calculations is done.
Because `finalizeBids()` is sent with an empty array and must wait for confirmation, **there is a time window where bidding is still open, yet the finalize transaction is visible in the public mempool**, enabling reactive bid updates.

<img width="1362" height="288" alt="Image" src="https://github.com/user-attachments/assets/c531d441-c9c8-4118-b2fd-a79ba690a652" />

It is important to note that if the calculations were done before finalize bid is called, it means that if the attacker front runs the call, they'll be at loss if its already calculated. But since its not then this is risk.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner submits the `finalizeBids()` transaction (with an empty array).
2. This transaction enters Polygon's public mempool.
3. Attacker sees the pending `finalizeBids()` and knows bidding will end shortly.
4. Attacker quickly submits new bids with higher gas fees.
5. Miner includes the attacker’s bids **before** the `finalizeBids()` transaction.
6. `finalizeBids()` confirms, but attacker’s last-second bids are already included.

### Impact

Bidding participants receive **unfair and manipulated allocations**, as attackers can always adjust their bids at the last second.
Refund amounts and TVS outcomes become **distorted**, defeating the purpose of hiding the endtime.

<img width="1453" height="512" alt="Image" src="https://github.com/user-attachments/assets/da5f43b2-b457-453d-ad79-1c4ab87d2344" />

### PoC

_No response_

### Mitigation

_No response_
  