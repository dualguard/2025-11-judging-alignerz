# [000073] setUpdateBidFee() maximum limit is 10x lower than it should
  
  ## Summary

The [`setUpdateBidFee()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L111) function enforces a maximum limit of 0.1 USDC (100,000 in 6 decimals), but the [README](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/README.md?plain=1#L17) documentation states the maximum should be 1 USDC (1,000,000 in 6 decimals). This 10x discrepancy results in a 90% protocol revenue loss on every bid update, as the protocol cannot collect fees at the intended rate.

## Vulnerability Details

### Code

```solidity
function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high"); // 0.1 USDC max

    uint256 oldUpdateBidFee = updateBidFee;
    updateBidFee = newUpdateBidFee;

    emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
}
```

### The Problem

- Code allows: `newUpdateBidFee < 100001` = 0.1 USDC (100,000 in 6 decimals)
- README states: Maximum of 1 USDC (1,000,000 in 6 decimals)
- Discrepancy: 10x difference

## Impact

- 90% revenue loss on every bid update
- Protocol can only collect 0.1 USDC max instead of intended 1 USDC per update
- Owner cannot set update bid fees as high as intended
- Requires contract upgrade to fix (no workaround available)

## Severity

### MEDIUM

Meets Medium Impact Criteria:

- Funds are indirectly at risk - Protocol fee revenue represents funds that are permanently limited to 10% of intended amount
- Partial disruption of functionality - Fee collection works but only at 10% of designed capacity

High Likelihood:

- Affects every bid update automatically
- No special conditions required
- Already impacts deployed contracts

## Recommended

```solidity
function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee <= 1_000_000, "Bid update fee too high"); // 1 USDC max

    uint256 oldUpdateBidFee = updateBidFee;
    updateBidFee = newUpdateBidFee;

    emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
}
```
  