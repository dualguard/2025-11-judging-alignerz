# [000155] Users can DoS the dividend distribution with excessive splits
  
  An attacker can use the split function in the vesting contract to create an enormous number of NFTs, and then use that large number to make the set up dividend function to stop working. 

The splitTVS function allows an user to take one NFT and split it into many. By repeatedly splitting the resulting NFTs in a series of separate, lower cost transactions, an attacker can exponentially increase the total supply of NFTs to tens of thousands without hitting gas limits on any single split transaction.

The distributor contract's setUpTheDividends function, which is required to start any dividend payout, contains loops that must iterate through every single minted NFT (for (uint i; i < nft.getTotalMinted();)). 

```solidity
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
```

As you can see above, each single iteration does an external call. Since the split can be called in multiple iterations, this attack is feasible on every single chain, including megaETH which has an excessively large gas limit.
  