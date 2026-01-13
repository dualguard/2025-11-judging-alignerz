# [000136] Decimal insensitive bid fee causes fee over/underpayment across stablecoins
  
  ### Summary

The bidding fee logic is not decimal-aware, while projects can use stablecoins with different decimals (e.g. 6‑decimals USDC vs 18‑decimals tokens like USDT). Because `bidFee`/`updateBidFee` are set contract‑wide and applied uniformly, users may significantly overpay or underpay fees depending on the stablecoin’s decimals. This is a **high severity** issue as it can cause direct loss of funds either for users or for the protocol.

### Root Cause

At `AlignerzVesting::placeBid` (line 729) and `AlignerzVesting::updateBid` (line 772), fee amounts are calculated and charged without normalizing for the `stablecoin`’s decimals configured per project. The same raw fee value is interpreted differently across 6‑ and 18‑decimal tokens.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L769-L770

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L727-L728



### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Protocol configures a project with a 6‑decimal stablecoin (e.g. USDC on Polygon) but keeps `bidFee` calibrated for 18‑decimal tokens.  
2. Users pay either 1e12x too much or too little per “unit” of fee, depending on how `bidFee` was chosen.  
3. Over time, this miscalibration drains users or leaves the protocol under‑compensated.


### Impact

Users may overpay fees by orders of magnitude, or the protocol may collect far less than intended, creating material, systematic value shifts.


### PoC

If `bidFee = 1e18` is intended as “1 token” for an 18‑decimal stablecoin, then on a 6‑decimal USDC it represents `1e12` USDC units (1,000,000,000,000 USDC), making fees catastrophically mis‑scaled.

### Mitigation

Consider normalizing fee amounts to a common 18‑decimal basis internally, or scaling `bidFee`/`updateBidFee` per project based on `stablecoin.decimals()`. Alternatively, store and apply fee rates as percentages of the bid amount, computed in a decimal‑safe way.
  