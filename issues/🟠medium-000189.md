# [000189] `_setBidFee` and `_setUpdateBidFee` functions allow the owner to set a fee up to 0.1$, 10 times less than what expected.
  
  ### Summary

`setBidFee` and `setUpdateBidFee` functions, defined in _FeesManager_ contract, are responsible for setting the bid fee and the update bid fee for the protocol.

They are defined as follows:

`setBidFee`:
```solidity
    function setBidFee(uint256 _bidFee) public onlyOwner {
        _setBidFee(_bidFee);
    }

    function _setBidFee(uint256 newBidFee) internal {
        require(newBidFee < 100001, "Bid fee too high");

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }
```

`setUpdateBidFee`:
```solidity
    function setUpdateBidFee(uint256 _updateBidFee) public onlyOwner {
        _setUpdateBidFee(_updateBidFee);
    }

    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
        require(newUpdateBidFee < 100001, "Bid update fee too high");

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }
```

https://github.com/dualguard/2025-11-alignerz/blob/main/README.md

Readme says "bidFee and updateBidFee can have a maximum of 1 USDC/USDT"

The problem arises because current implementation is incorrect, with one zero missing to allow the fee to be set to up to 1 USDC/USDT.


### Root Cause

The root cause lies in the incorrect check in both `setBidFee` and `setUpdateBidFee` functions:
```solidity
        require(newBidFee < 100001, "Bid fee too high");
```
```solidity
        require(newUpdateBidFee < 100001, "Bid update fee too high");
```

The value `100001` should be `1000001` because USDC and USDT have 6 decimals, not 5.

This means current implementation only allows a fee up to 0.1 USDC/USDT.

### Impact

The impact of this issue is medium as it results in a fee being potentially 10 times less than what the owner would expect. This is a significant loss of potential treasury rewards.

### Mitigation

Make sure to correctly implement `setBidFee` and `setUpdateBidFee` functions:

``` solidity
    function setBidFee(uint256 _bidFee) public onlyOwner {
        _setBidFee(_bidFee);
    }

    function _setBidFee(uint256 newBidFee) internal {
        // Example placeholder limits
        require(newBidFee < 1000001, "Bid fee too high");

        uint256 oldBidFee = bidFee;
        bidFee = newBidFee;

        emit bidFeeUpdated(oldBidFee, newBidFee);
    }

    function setUpdateBidFee(uint256 _updateBidFee) public onlyOwner {
        _setUpdateBidFee(_updateBidFee);
    }

    function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
        require(newUpdateBidFee < 1000001, "Bid update fee too high");

        uint256 oldUpdateBidFee = updateBidFee;
        updateBidFee = newUpdateBidFee;

        emit updateBidFeeUpdated(oldUpdateBidFee, newUpdateBidFee);
    }
  