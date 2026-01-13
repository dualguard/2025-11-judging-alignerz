# [000826] Fee Manager's bid fee and update bid fee are checked against the wrong constant
  
  ### Summary

Fee Manager checks bid fee and update bid fee against the wrong constant of 0.1 USDC rather than 1 USDC.


### Root Cause


According to the contest README:

> `bidFee` and `updateBidFee` can have a maximum of 1 USDC/USDT

However, the upper bound of these fees are being checked against the wrong constant:

```solidity
 function _setBidFee(uint256 newBidFee) internal {
    // ...
    require(newBidFee < 100001, "Bid fee too high"); // @audit wrong constant

    // ...
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L84-L92

```solidity
function _setUpdateBidFee(uint256 newUpdateBidFee) internal {
    require(newUpdateBidFee < 100001, "Bid update fee too high");
    // ...
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L110-L117

USDC and USDT have 6 decimals, so 1 token is denoted by a constant of `1000000`, not `100000`. Then the maximum fee that can be set is only 0.1 USDC/USDT.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Admin cannot set the bid fee/update bid fee to 1 USDC as desired as attempting it will revert


### Impact

Admin cannot set the bid fee/update bid fee to 1 USDC as desired. This causes loss of bid fee.


### PoC

_No response_

### Mitigation

Change the constant to `1_000_001`

  