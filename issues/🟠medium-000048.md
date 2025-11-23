# [000048] Owner Controlled Merge Fee Conflicts with Whitepaper Statement
  
  ### Summary

The whitepaper, which serves as the source of truth, explicitly states that merging TVS should be “free of charge.” Specifically:

> **Splitter Fee:** 0.5% of the TVS value, automatically discounted from the tokens.
> **Merging Fee:** Free of charge.                                       
    

The contract contradicts this statement by allowing the owner to set a custom merge fee through the `setMergeFeeRate()` function. The `mergeFeeRate` variable can be set to any value below 201 basis points. This behavior deviates from the whitepaper, which users rely on as the definitive guide for fees.

### Root Cause

```solidity
 /**
     * @notice Updates the merge fee.
     * @param _mergeFeeRate The new merge fee value.
     *
     * Emits a {mergeFeeRateUpdated} event.
     */
    function setMergeFeeRate(uint256 _mergeFeeRate) public onlyOwner {
        _setMergeFeeRate(_mergeFeeRate);
    }

    /**
     * @notice Internal function to update the merge fee.
     * @param newMergeFeeRate The new merge fee value.
     *
     * Emits a {mergeFeeRateUpdated} event.
     */
    function _setMergeFeeRate(uint256 newMergeFeeRate) internal {
        require(newMergeFeeRate < 201, "Merge fee too high");

        uint256 oldMergeFeeRate = mergeFeeRate;
        mergeFeeRate = newMergeFeeRate;

        emit mergeFeeRateUpdated(oldMergeFeeRate, newMergeFeeRate);
    }
```
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L150

### Impact

- The system violates user trust, as the owner can override what the whitepaper specifies.
    
- Auditors or users referencing the whitepaper will encounter inconsistent behavior leading to disputes.

### PoC

Similar issues that are inconsistent with the whitepaper are classified as Medium severity.

[Sherlock](https://solodit.cyfrin.io/issues/m-3-the-default-value-of-epsilon-differs-from-what-is-stated-in-the-whitepaper-sherlock-allora-git)
[Code4rena](https://solodit.cyfrin.io/issues/m-02-blocked-accounts-keep-earning-interest-contrary-to-the-whitepaper-code4rena-wildcat-protocol-wildcat-protocol-git)
[Sherlock](https://solodit.cyfrin.io/issues/m-4-the-formula-for-forecast-normalization-differs-from-the-one-in-the-whitepaper-sherlock-allora-git)


### Mitigation

Update the whitepaper if the merge fee is intended to be adjustable or modify the code to align with the whitepaper
  