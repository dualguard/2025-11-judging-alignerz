# [000561] Repeated External Calls in `getUnclaimedAmounts()` Cause Excessive Gas Consumption and Out-of-Gas Failures
  
  

## Summary

Repeated calls to `vesting.allocationOf(nftId)` in `getUnclaimedAmounts()` will cause out-of-gas failures for the protocol and users as the protocol owner cannot execute dividend distribution functions when the NFT collection grows, due to excessive gas consumption from making 6 external calls per NFT instead of caching the result.

&nbsp;

## Root Cause

In [`A26ZDividendDistributor.sol:140-146`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141-L146), the `getUnclaimedAmounts()` function calls `vesting.allocationOf(nftId)` six separate times to access different fields of the returned `Allocation` struct:
```js
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
		// ...
	}
```

Since `getTotalUnclaimedAmounts()` (lines 127-136) iterates over all NFTs and calls `getUnclaimedAmounts()` for each owned NFT, this results in 6 external calls per NFT. Each external call has overhead (cross-contract call gas), so this multiplies gas usage. With many NFTs, the total gas can exceed the block gas limit, causing out-of-gas failures.

&nbsp;

## Internal Pre-conditions

1. The number of minted NFTs (`nft.getTotalMinted()`) needs to be sufficiently large such that the total gas cost (6 external calls × number of owned NFTs) approaches or exceeds the block gas limit
2. The protocol owner needs to call `setUpTheDividends()`, `setAmounts()`, or any function that internally calls `getTotalUnclaimedAmounts()` to trigger the vulnerability

&nbsp;

## External Pre-conditions

None required. The vulnerability manifests naturally as the protocol grows or can be intentionally triggered by an attacker who splits NFTs to increase the total count.

&nbsp;

## Attack Path

1. As the protocol grows, the number of minted NFTs increases through normal operations or through an attacker repeatedly splitting NFTs
2. When the protocol owner attempts to set up dividend distributions by calling `setUpTheDividends()`, it internally calls `_setAmounts()`
3. `_setAmounts()` calls `getTotalUnclaimedAmounts()` (line 209)
4. `getTotalUnclaimedAmounts()` iterates over all NFTs from `0` to `nft.getTotalMinted()` (lines 128-135)
5. For each owned NFT, it calls `getUnclaimedAmounts(i)` (line 131)
6. `getUnclaimedAmounts()` makes 6 separate external calls to `vesting.allocationOf(nftId)` for each NFT (lines 141-146)
7. With a large number of NFTs (e.g., 1,000+), this results in thousands of external calls (6 × number of owned NFTs)
8. The cumulative gas cost exceeds the block gas limit, causing the transaction to revert with an out-of-gas error
9. The protocol owner cannot execute dividend distribution functions, and TVS holders cannot receive their dividends

&nbsp;

## Impact

The protocol and TVS holders cannot execute dividend distribution functions. The protocol owner cannot set up or update dividend distributions due to out-of-gas failures, and TVS holders cannot claim dividends. This occurs naturally as the protocol scales or can be intentionally triggered by an attacker through NFT splitting. 

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Cache the result of `vesting.allocationOf(nftId)` in a local variable and reuse it:

```js
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    Allocation memory allocation = vesting.allocationOf(nftId);
    
    if (address(token) == address(allocation.token)) return 0;
    
    uint256[] memory amounts = allocation.amounts;
    uint256[] memory claimedSeconds = allocation.claimedSeconds;
    uint256[] memory vestingPeriods = allocation.vestingPeriods;
    bool[] memory claimedFlows = allocation.claimedFlows;
    uint256 len = amounts.length;
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

This reduces external calls from 6 per NFT to 1 per NFT, reducing gas consumption by approximately 83% and preventing out-of-gas failures as the protocol scales.


  