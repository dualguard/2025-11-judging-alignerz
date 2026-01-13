# [000333] Admin will be unable to set bidFee and updateBidFee to documented maximum values, resulting in a 90% loss on gains for the protocol
  
  ### Summary

Incorrect validation bounds in fee setters will cause the admin to be unable to set `bidFee` and `updateBidFee` to their documented maximum of 1 USDC/USDT, as the checks only allow up to 0.1 USDC/USDT (100,000 units instead of 1,000,000 units with 6 decimals).


### Root Cause

In [FeesManager.sol:85](https://github.com/dualguard/2025-11-alignerz/blob/main/src/FeesManager.sol#L85) and [FeesManager.sol:106](https://github.com/dualguard/2025-11-alignerz/blob/main/src/FeesManager.sol#L106), the validation checks use `< 100001` (allowing maximum 100,000) instead of `< 1000001` (allowing maximum 1,000,000). Since USDC/USDT use 6 decimals, this limits the fees to 0.1 USDC/USDT instead of the documented 1 USDC/USDT maximum stated in the [README](https://github.com/dualguard/2025-11-alignerz/blob/main/README.md?plain=1#L17): ``bidFee` and `updateBidFee` can have a maximum of 1 USDC/USDT`


### Internal Pre-conditions

1. Admin needs to call `setBidFee()` or `setUpdateBidFee()` with values between 100,001 and 1,000,000 to set fees above 0.1 USDC/USDT.

Funds are at risk 


### External Pre-conditions

None

### Attack Path

This is not an attack path but a **constraint violation** (degraded functionality):

1. Admin attempts to call `setBidFee(500000)` to set fee to 0.5 USDC.
2. Transaction reverts with "Bid fee too high".
3. Admin is forced to set a maximum of 0.1 USDC instead of the intended 0.5 USDC (or expected maximum 1 USDC).


### Impact

The protocol cannot set fees to their documented maximum values, resulting in up to 90% less fee revenue than intended (0.1 USDC actual vs 1 USDC intended). This directly contradicts the protocol's documented fee structure and may impact the protocol's economic sustainability.

Also, 0.1 USDC is probably less than the gas cost of sending them from the function. It is unsustainably low

Given the high impact:
```
• Funds are directly or almost directly at risk
• Severe disruption of protocol functionality or availability
```

And also 100% likelihood : the chosen severity is High.

### PoC

A coded POC would be a bit trivial just to show a missing `0`. Instead, please see the following diff, and count the 0s (with in mind that USDC/USDT are 6 decimals so 1 USDC is 1e6 meaning 1 million, and here the max is a hundred thousand):
```diff
File: FeesManager.sol

    function _setBidFee(uint256 newBidFee) internal {
-       require(newBidFee < 100001, "Bid fee too high"); // @audit allows max 0.1 USDC/USDT
+       require(newBidFee < 1000001, "Bid fee too high"); // @audit allows max 1 USDC/USDT

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }

    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
-       require(newUpdateBidFee < 100001, "Bid update fee too high"); // @audit allows max 0.1 USDC/USDT
+       require(newUpdateBidFee < 1000001, "Bid update fee too high"); // @audit allows max 1 USDC/USDT

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }
```
https://github.com/dualguard/2025-11-alignerz/blob/main/README.md?plain=1#L17 : 

<img width="850" height="115" alt="Image" src="https://github.com/user-attachments/assets/e311fa5f-368e-47fa-8a94-82abdb19af7f" />

### Mitigation

Change the validation bounds from `100001` to `1000001` in both `_setBidFee()` and `_setUpdateBidFee()` to align with the documented 1 USDC/USDT maximum.
  