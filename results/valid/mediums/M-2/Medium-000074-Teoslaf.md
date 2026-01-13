# [000074] setBidFee() maximum limit is 10x lower than it should
  
  ## Summary

The [`setBidFee()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L86) function enforces a maximum limit of 0.1 USDC (100,000 in 6 decimals), but the [README](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/README.md?plain=1#L17) documentation states the maximum should be 1 USDC (1,000,000 in 6 decimals). This 10x discrepancy results in a 90% protocol revenue loss on every bid placement, as the protocol cannot collect fees at the intended rate.

## Vulnerability Details

### Code

```solidity
function _setBidFee(uint256 newBidFee) internal {
    require(newBidFee < 100001, "Bid fee too high"); // 0.1 USDC max

    uint256 oldBidFee = bidFee;
    bidFee = newBidFee;

    emit bidFeeUpdated(oldBidFee, newBidFee);
}
```


## Impact

### Financial Impact

- 90% revenue loss on every bid placement
- Protocol can only collect 0.1 USDC max instead of intended 1 USDC per bid
- Owner cannot set bid fees as high as intended
- Funds indirectly at risk - Protocol treasury receives 90% less fee revenue than designed

## Severity

### Medium

- Funds are indirectly at risk - Protocol fee revenue represents funds that are permanently limited to 10% of intended amount
- Partial disruption of functionality - Fee collection works but only at 10% of designed capacity

High Likelihood:

- Affects every bid placement automatically
- No special conditions required
- Already impacts deployed contracts


## Recommended

```solidity
function _setBidFee(uint256 newBidFee) internal {
    require(newBidFee <= 1000000, "Bid fee too high"); // 1 USDC max

    uint256 oldBidFee = bidFee;
    bidFee = newBidFee;

    emit bidFeeUpdated(oldBidFee, newBidFee);
}
```

  