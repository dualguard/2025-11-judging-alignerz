# [000576] Flat `updateBidFee` Value Without Decimal Normalization Causes Inconsistent Fee Charges Across Different Stablecoin Decimals and Blockchains
  
  
## Summary

The use of a flat `updateBidFee` value without accounting for stablecoin decimals causes inconsistent fee charges for users across chains. Users pay different dollar amounts depending on the stablecoin decimals on Polygon, Arbitrum, and MegaETH, leading to overpayment or underpayment relative to the intended fee.

## Root Cause

In [`AlignerzVesting.sol:770-771`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L769-L771), `updateBidFee` is used directly without normalizing for the stablecoin’s decimals. The fee is stored as a raw `uint256` in `FeesManager.sol:12` and transferred directly via `safeTransferFrom` without adjusting for the token’s decimal places.

The design assumes all supported stablecoins use the same decimals. USDC and USDT on Polygon and Arbitrum use 6 decimals, but if MegaETH launches with 18-decimal tokens, the same `updateBidFee` value will represent different dollar amounts.

```js
    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
        require(newUpdateBidFee < 100001, "Bid update fee too high");

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }
```

Additionally, in `FeesManager.sol:110-111`, there is a hard constraint that limits `updateBidFee` to be less than `100,001` raw units. This constraint compounds the issue: if the fee is set assuming 6-decimal tokens (e.g., `100,000` = 0.1 USDC), it cannot be scaled appropriately for 18-decimal tokens, where `100,000` would represent only `0.0000000001` USDC. Conversely, if set for 18-decimal tokens, the constraint prevents setting a value high enough to work correctly on 6-decimal tokens.

For example, if `updateBidFee = 1_000_000` (intended as 1 USDC for 6-decimal tokens):
- On Polygon/Arbitrum: 1_000_000 = 1.0 USDC
- On MegaETH (18 decimals): 1_000_000 = 0.000001 USDC

However, due to the constraint, the maximum value is `100,000`, which means:
- On Polygon/Arbitrum: 100,000 = 0.1 USDC (maximum possible)
- On MegaETH (18 decimals): 100,000 = 0.0000000001 USDC (negligible)

## Internal Pre-conditions

1. Admin must call `setUpdateBidFee()` to set `updateBidFee` to a specific raw value
2. The `updateBidFee` value must be less than `100,001` due to the hard constraint in `_setUpdateBidFee()`
3. The protocol must support stablecoins with different decimal configurations across chains
4. A bidding project must be created with a stablecoin that has decimals different from the assumed decimals when `updateBidFee` was set

## External Pre-conditions

1. USDC or USDT deployed on MegaETH must use decimals other than 6 (e.g., 18)
2. The protocol must be deployed on MegaETH with stablecoins having different decimals than Polygon/Arbitrum

## Attack Path

1. Admin sets `updateBidFee` to `100,000` (maximum allowed, intended as 0.1 USDC for 6-decimal tokens)
2. Protocol is deployed on MegaETH where USDC/USDT use 18 decimals
3. User calls `updateBid()` on MegaETH
4. Contract transfers `100,000` raw units, which equals `0.0000000001` USDC instead of `0.1` USDC
5. User pays a negligible fee (1 billionth of the intended amount), causing significant protocol revenue loss

Alternatively:
1. Admin sets `updateBidFee` to `100,000` assuming 18-decimal tokens (intended as 0.1 USDC)
2. User calls `updateBid()` on Polygon/Arbitrum where tokens use 6 decimals
3. Contract transfers `100,000` raw units, which equals `0.1` USDC (correct in this case, but incorrect if the admin intended it for 6-decimal tokens)

## Impact

Protocol suffer financial loss due to incorrect fee calculations. If `updateBidFee` is set for 6 decimals (e.g., `100,000` = 0.1 USDC) but used on 18-decimal tokens, users pay `0.0000000001` USDC instead of `0.1` USDC, causing a 1 billion-fold reduction in protocol revenue. The hard constraint of `100,001` prevents setting higher values to compensate for decimal differences, making it impossible to set a fee that works correctly across all decimal configurations. The exact impact depends on the decimal mismatch and the set fee value, but the constraint ensures the maximum loss is bounded by the cap.

## Proof of Concept
N/A

## Mitigation

Normalize `updateBidFee` based on the stablecoin’s decimals when charging the fee.
  