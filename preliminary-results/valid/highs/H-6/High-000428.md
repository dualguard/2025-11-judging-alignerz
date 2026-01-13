# [000428] ``splitTVS()`` can be used by malicious actors to permanently break the ``dividend`` distribution mechanism.
  
  ### Summary

``splitTVS()`` can be used by malicious actors to permanently break the ``dividend`` distribution mechanism.

### Root Cause

``setAmounts()`` and ``setDividends()`` are very gas heavy as they loop over all the ``nfts`` ever minted with ``nested`` loops in between.
```solidity
./A26zDividendDistibutor.sol

    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```
All the gas heavy functions:
1. ``_setDividends()``: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224
2. ``getUnclaimedAmounts()``: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161
3. ``_setAmounts()``: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211

These functions alone may be enough and consume all the ``block gas limit``. But a malicious actor can take it ``further``.
1. The bidder at the top of the ``first`` and ``last`` pool will get refunded. And bidders are ordered by vesting period lengths. A malicious actor ``Alice`` can buy exactly ``10000`` tokens + ``splitFeeRate`` amount of tokens from ``first pool`` by setting ``vesting lengths to (uint256).max / 2_592_000 * 2_592_000``. ``Alice`` will be the top bidder as nobody will top that. ``Alice`` can do this by sending a ``last second`` transcation before the ``bidding end time``. 
> Bidding end time although hashed can be easily read from the ``launchBiddingProject()`` transcation.
3. ``Alice`` will call claim the ``TVS NFT`` and split that into ``10000`` NFTs using the ``splitTVS()`` function . As there is no percentage limit and ``sumOfPercentages need to be equal to BASIS_POINT(10000)``  at the end. ``splitTVS()``: https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107
3. ``Alice`` gets refunded due to being the top bidder and can carry out step 1 and 2 until ``Dividend Distribution`` mechanism completely breaks due to huge number of ``tokens``.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Mentioned in the Root Cause.

### Impact

Permanently breaking the core functionality of the protocol.

### PoC

_No response_

### Mitigation

1. Add minimum ``split`` percentage and ``max limit``.
2. Refactor the ``dividend distribution`` mechanism.
  