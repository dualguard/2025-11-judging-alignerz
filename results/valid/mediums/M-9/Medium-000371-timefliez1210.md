# [000371] H07 DoS in A26ZDividendDistributor due to O(NM) complexity
  
  ## Summary

The `A26ZDividendDistributor` contract contains a nested loop structure within `getTotalUnclaimedAmounts` and `_setDividends` that iterates over all minted NFTs and their respective vesting flows. This $O(N \times M)$ complexity ensures that as the number of NFTs grows, the gas cost to execute these functions will exceed the block gas limit, causing a permanent Denial of Service (DoS) for dividend distribution.

## Root Cause

In `A26ZDividendDistributor.sol`:

```solidity
    function getTotalUnclaimedAmounts()
        public
        returns (uint256 _totalUnclaimedAmounts)
    {
        // ...
        uint256 len = nft.getTotalMinted();
1. @>   for (uint i; i < len; ) {
            (, bool isOwned) = safeOwnerOf(i);
2. @>       if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

And `getUnclaimedAmounts`:

```solidity
    function getUnclaimedAmounts(
        uint256 nftId
    ) public returns (uint256 amount) {
        // ...
        uint256 len = vesting.allocationOf(nftId).amounts.length;
3. @>   for (uint i; i < len; ) {
            // ...
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

1.  **Outer Loop (1)**: Iterates $N$ times, where $N$ is the total number of minted NFTs.
2.  **Inner Loop (3)**: Called at (2), iterates $M$ times, where $M$ is the number of vesting flows for the specific NFT.

The total complexity is $O(N \times M)$. Additionally, `getUnclaimedAmounts` performs multiple external calls to `vesting.allocationOf(nftId)`, which copies large arrays from storage to memory, further increasing gas costs.

## Internal Pre-Conditions

1.  A sufficient number of NFTs must be minted ($N$).
2.  NFTs must have vesting allocations ($M$).

## External Pre-Conditions

None.

## Attack Path

1.  The protocol operates normally until the number of NFTs reaches a threshold where the gas cost of `getTotalUnclaimedAmounts` exceeds the block gas limit.
2.  The owner calls `setDividends` or `setUpTheDividends`.
3.  The transaction reverts due to Out of Gas.
4.  Dividends can never be distributed again.

## Impact

Permanent Denial of Service of the core dividend distribution functionality. Funds locked in the contract intended for dividends cannot be distributed. High Likelihood (inevitable with growth) and High Impact.

## PoC

Trivial.

## Mitigation

Refactor the dividend distribution mechanism to avoid iterating over all NFTs in a single transaction.
1.  **Pull over Push**: Allow users to claim dividends individually, calculating their share based on their own holdings at the time of claim.
2.  **Batched Distribution**: If push is required, allow processing in batches (e.g., process 50 NFTs at a time).

Second solution might be very much a desired choice, since this loop is only used for a number of known NFTs from the Alignerz sale.

  