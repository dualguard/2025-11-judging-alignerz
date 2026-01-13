# [000149] Missing extra refund feature for winners
  
  The hasExtraRefund variable is useless because it's never used in any contract logic. Based on its name and the comments on the struct:

```solidity
    struct VestingPool {
        bytes32 merkleRoot; // Merkle root for allocated bids
        bool hasExtraRefund; // whether pool refunds the winners as well
    }
```

Its purpose was to allow users in certain pools to both claim their winning token allocation and also receive a partial refund, a feature that was never implemented. This feature has not been implemented at any point, and this variable is merely assigned to the vesting struct and emitted at pool creation.
  