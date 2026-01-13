# [000111] Hardcoded fee caps in FeesManager.sol will cause an inability to set correct fees for the protocol owner
  
  ### Summary

Hardcoded fee caps in FeesManager.sol will cause an inability to set correct fees for the protocol owner as the cap is too low for standard 18-decimal tokens.


### Root Cause

In src/contracts/vesting/feesManager/FeesManager.sol, the function _setBidFee enforces require(newBidFee < 100001). For an 18-decimal token (e.g., DAI), 100001 is 0.0000000000001 DAI.


### Internal Pre-conditions

1-The protocol uses a standard ERC20 token with 18 decimals for bidding.

### External Pre-conditions

None.

### Attack Path

1-Owner attempts to set the bid fee to 1 DAI (1000000000000000000).
2-Contract reverts with "Bid fee too high".

### Impact

The protocol cannot enforce meaningful fees for the IWO process if standard tokens are used.


### PoC

```solidity
// See test_PoC_7_FeeCap_RevertsOnStandardValues in AlignerzAdditionalPoC.t.sol
function test_PoC_7_FeeCap_RevertsOnStandardValues() public {
    uint256 standardFee = 1 ether; 
    vm.expectRevert("Bid fee too high");
    vesting.setBidFee(standardFee);
}
```

### Mitigation

Remove the hardcoded numeric cap or scale the cap dynamically based on the token's decimals.

  