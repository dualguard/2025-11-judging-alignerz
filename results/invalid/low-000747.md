# [000747] Flat bid fees are not normalized to token decimals, so using low decimals stables makes the caps meaningless
  
  
### Summary

The bid and update fees are stored as raw token amounts and capped with a hard‑coded `100001` limit, which implicitly assumes a 6‑decimals stablecoin; if the owner switches to a 2‑decimals stable like GUSD, the same numeric cap allows fees more than 1000 tokens, far beyond the intended range.

### Root Cause

In `FeesManager` the flat fees are bounded without considering `decimals()`:

```solidity
function _setBidFee(uint256 newBidFee) internal {
    require(newBidFee < 100001, "Bid fee too high");
    ...
}

function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high");
    ...
}
```

- With a 6‑decimals stable, `100001` ≈ `0.100001` tokens - reasonable cap.  
- With a 2‑decimals stable (e.g. GUSD), `100001` ≈ `1000.01` tokens - very large cap.  

The protocol lets the trusted owner change `biddingProject.stablecoin` freely, but the fee caps do not adapt to differing decimals, so the same numerical limit allows extremely high effective fees when a low‑decimals stable is selected.

### Impact

If the owner switches to a low‑decimals token and reuses the same fee configuration assumptions, users can end up paying much larger absolute fees than intended because the contract does not normalize by decimals.

### Mitigation

Normalize flat fees to stables’ decimals or remove the hard `100001` magic number:

- Prefer percentage‑based fees (BPS of `amount`) instead of raw token units for `bidFee` / `updateBidFee`, or  
- Store per‑token configuration that includes `decimals()` and define the max fee in human units (e.g. “max 0.1 stable”) rather than a fixed raw limit.
  