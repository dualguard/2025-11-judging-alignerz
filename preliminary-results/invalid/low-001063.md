# [001063] Missing Commit–Reveal Validation Allows Bidding Window Hash to Serve No Security Purpose
  
  ### Summary

The documentation states that Alignerz uses a commit–reveal scheme to prevent last-minute bid manipulation. The owner is expected to commit a hashed bidding window (endTimeHash) during `launchBiddingProject()` and later reveal the preimage during `finalizeBids()` to prove that bidding ended at the predetermined moment.

However, the implementation never validates this commit–reveal process. `finalizeBids()` does not accept a reveal value, does not verify the hash, and never enforces that bidding ended at the committed time. As a result, the hashing mechanism does not enforce fairness and becomes a non-functional, unused feature.

> To prevent last-minute bid manipulation, bidding window will be hashed.
Ps: bidding window is a period of time that starts as soon as the pools are filled.

> After the bidding ends, we will reveal the exact input used to generate the hash, ensuring
fairness.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L656

When creating a bidding project the above function stores `endTimeHash`.

But the pre-image of `endTimeHash` is not provided during `finalizeBids`. The below is reference for `finalizeBids` function. It does not accept a reveal value, compute hash value.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783

### Internal Pre-conditions

None

### External Pre-conditions

1. Documentation promises a commit–reveal bidding window to prevent last-minute manipulation.
2. Users rely on this mechanism for fairness assumptions.

### Attack Path

NA

### Impact

The commit–reveal scheme advertised for fairness is non-functional, meaning users cannot rely on the documented protection against last-minute manipulation. Although this does not create an exploit by itself, it:
- Removes a security guarantee promised by the protocol.
- Misleads users regarding bidding fairness.
- Reduces trust and transparency of the IWO bidding model.

### PoC

_No response_

### Mitigation

_No response_
  