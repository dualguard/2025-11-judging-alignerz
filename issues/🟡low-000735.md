# [000735] Hardcoded Fee Limit Prevents Setting Economically Meaningful Bid Fees And Might Lead To Spam
  
  ### Summary

The hardcoded limit of `100001` in `FeesManager::_setBidFee()` and `FeesManager::_setUpdateBidFee()` prevents the protocol owner from setting economically meaningful bid fees for USDT and USDC (6-decimal stablecoins). The maximum allowed fee of 100000 units equals only 0.1 USDC/USDT (10 cents), making the fee mechanism ineffective for spam prevention or revenue generation.

### Root Cause

The hardcoded validation `require(newBidFee < 100001, "Bid fee too high")` and `require(newUpdateBidFee < 100001, "Bid update fee too high")` fail to account for token decimals. Since the protocol uses 6-decimal stablecoins (USDT and USDC) for bidding, this limit restricts fees to a maximum of 0.1 tokens. 
```solidity
 function _setBidFee(uint256 newBidFee) internal {
        // Example placeholder limits
        require(newBidFee < 100001, "Bid fee too high"); //@> only 0.1 usdc can cause spam

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }
```

```solidity
  function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
        require(newUpdateBidFee < 100001, "Bid update fee too high"); //@> only 0.1 usdc can cause spam

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }

```

### Internal Pre-conditions

1. Owner needs to call `FeesManager::setBidFee()` or `FeesManager::setUpdateBidFee()` to set fees above 100000 units (e.g., 1000000 for 1 USDC).
2. Protocol uses USDT or USDC (6-decimal tokens) as the stablecoin for bidding projects.

### External Pre-conditions

None

### Attack Path

1. Owner attempts to set a meaningful bid fee (e.g., 1 USDC = 1000000 units) by calling `setBidFee(1000000)`
2. Transaction reverts with `"Bid fee too high"` because 1000000 >= 100001
3. Owner is forced to set fee ≤ 100000, which equals only 0.1 USDC.
4. Users can spam bids with minimal cost, and the protocol cannot generate meaningful revenue from bidding fees.

### Impact

The protocol owner cannot set economically meaningful bid fees for spam prevention or revenue generation. With the maximum fee capped at 0.1 USDC/USDT, users can place unlimited bids at negligible cost, defeating the purpose of the fee mechanism.

### PoC

_No response_

### Mitigation

Remove or significantly increase the hardcoded limits in `FeesManager::_setBidFee()` and `FeesManager::_setUpdateBidFee() `to accommodate 6-decimal stablecoins:
  