# [000613] Malicious user can honeypot NFT buyers and claim their rewards/stables before the sale occurs
  
  ### Summary

Users can sell their NFTs on markets. There's no measure to stop sellers from claiming their vested tokens right before the sale goes through. A malicious user can honeypot buyers - list the NFT when it's almost fully vested, then frontrun the buyer's purchase transaction to claim the available tokens, leaving the buyer with a worthless NFT.

Note: A similar bug was observed previously, refer to this report for comparison, the bug I refer is H-04 ⇒ https://codehawks.cyfrin.io/c/2023-12-the-standard/results?t=report 

### Details

Scenario:

- Alice gets NFT from successful bid - 100k tokens, 1 year vesting
- Day 364: Alice lists NFT for sale at 2 ETH
- NFT shows 99,700 tokens claimable (99.7% vested)
- Bob sees the listing: "good deal, almost fully vested"
- Bob sends transaction to buy the NFT

The attack:

- Alice frontruns with claimTokens() - claims 99,700 tokens
- Bob's purchase transaction executes - he pays 2 ETH
- Bob now owns NFT with only 300 tokens left 

Bob paid for 99.7k tokens, got 300. He got scammed.

Take into account that claiming doesn't burn the NFT until 100% => https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L969

```solidity
       if (flowsClaimed == nbOfFlows) {
            nftContract.burn(nftId);  // only burns when fully claimed
            allocation.isClaimed = true;
        }
```

After claiming 99.7%, NFT still exists with 0.3% value.
Marketplace doesn't know about claims in real-time - Bob sees stale data showing 99.7k claimable.

### Impact
High severity. Buyer overpays massively for almost-empty NFT. Easy to execute, profitable for attacker, costs only gas fees.

### Recommendation

Consider implementing a mechanism where the owner of the NFT has to pause all interactions regarding claims when they put the NFT on sale
  