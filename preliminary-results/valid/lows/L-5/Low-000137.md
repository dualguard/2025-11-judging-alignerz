# [000137] The prediction markets exact end time can realistically be calculated beforehand and used to change vesting length and amount last minute
  
  ### Summary

The prediction market’s “hidden” end time is derived from a very small and predictable input space and is only protected by a single hash. An attacker can precompute a significant fraction of all possible hashes once and then instantly recover the exact end time for any bidding round, allowing last‑minute manipulation of vesting length and committed amounts. This is a **high severity** issue because it breaks the intended unpredictability of the end time and directly enables timing‑based economic manipulation.

### Root Cause

The protocol uses a hash of a low‑entropy end time (e.g. a specific second within a constrained interval) as a commitment. The total number of possible end times is around 5.9x10^15, which is large but not cryptographically secure when an attacker can precompute a sizeable subset once and then use O(1) lookups.


### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

1. Offline, the attacker generates all candidate input strings for a chosen fraction (e.g. 5%) of the full end‑time space and stores `keccak256(input)` → `input` in a hash map.  
2. On-chain, when a new bidding project is launched, the attacker reads the published `endTimeHash`.  
3. The attacker performs a constant‑time lookup in the precomputed table and recovers the exact second at which bidding will stop.  
4. Knowing the exact stop time, the attacker adjusts their vesting period and bid size at the very end of the auction to front‑run or snipe honest users.


### Impact

The end time is no longer unpredictable for a determined attacker. This allows for precise last-second bidding and vesting manipulation, undermining fairness and potentially leading to systematically better outcomes for the attacker compared to honest participants. This breaks a core functionality of the protocol and can lead to users unfairly losing bidding wars (loss of funds)

### PoC

If the total space is 5.9x10^15 possible end times and a modern cluster can brute force the full space in ~1–3 days, then precomputing 5% ~3x10^14 inputs once is still practically feasible for a well‑resourced attacker. After that, each new `endTimeHash` can be resolved to its exact end time with a single hash‑table lookup.

The actual number of combinations is likely even smaller, since anything lower than a few minutes doesn't really make sense, as users need time to bid. 

### Mitigation

Consider increasing the entropy of the commitment by including microseconds or additional unpredictable components (e.g. a server‑side or oracle‑provided nonce) so that the effective input space is cryptographically large. Alternatively, adopt a more robust commit‑reveal scheme or VRF‑based mechanism where the end time is derived from on-chain randomness revealed only after commitments are fixed. The current industry standard is 2^{256} bit encryption, which is many magnitudes larger. 

  