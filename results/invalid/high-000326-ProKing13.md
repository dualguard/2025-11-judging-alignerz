# [000326] Missing Extra Refund Feature for Pool Winners
  
  ### Summary


The `AlignerzVesting` contract declares a `hasExtraRefund` boolean flag in the `VestingPool` [struct](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L57) that can be set during pool [creation](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L679) , but this flag is never used anywhere in the contract logic. The whitepaper explicitly promises that "first pool & last pool winners will keep their allocation & get refunded," meaning Pool 0 and the last pool winners should receive their tokens for free. However, no function exists to claim these extra refunds, and no logic checks or uses the `hasExtraRefund` flag.

From Whitepaper: ```We also decided to add additional incentives such as first pool & last pool winners will keep their
allocation & get refunded.```


### Root Cause


The root cause is incomplete feature implementation. The `hasExtraRefund` flag was added to the `VestingPool` struct and can be set during pool creation, but the development team never implemented the corresponding claim logic.

```javascript
struct VestingPool {
    bytes32 merkleRoot;
    bool hasExtraRefund;  // Flag exists but never used
}
```

**Flag Assignment:**
```javascript
function createPool(..., bool hasExtraRefund) external onlyOwner {
    biddingProject.vestingPools[poolId] = VestingPool({
        merkleRoot: bytes32(0),
        hasExtraRefund: hasExtraRefund  // Flag is stored
    });
}
```

**Missing Usage:** No function in the entire contract reads or checks this flag. The `claimNFT()` function allows winners to claim their allocation but has no logic to trigger refunds for pools with `hasExtraRefund = true`. No `claimExtraRefund()` function exists.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Thiis is not an exploitable vulnerability but a missing feature:

**Step 1:** Owner creates Pool 0 with extra refund enabled
```javascript
createPool(projectId: 0, totalAllocation: 2_200_000, tokenPrice: 0.10, hasExtraRefund: true);
// Pool 0 created, hasExtraRefund flag set to true
```

**Step 2:** Alice reads whitepaper and sees the promise
```
"Pool 1 Discount = 80% + Full USDT refund = More tokens & for free"
```

**Step 3:** Alice bids 220,000 USDT expecting to get it refunded
```javascript
placeBid(projectId: 0, amount: 220_000 USDT, vestingPeriod: 365 days);
// Alice commits 220,000 USDT
```

**Step 4:** Bidding ends, Alice wins allocation in Pool 0

**Step 5:** Alice claims her NFT
```javascript
claimNFT(projectId: 0, poolId: 0, amount: 2_200_000, merkleProof);
// Alice receives NFT with 2,200,000 A26Z tokens vesting
```

**Step 6:** Alice tries to claim her refund
```javascript
// Option A: Try claimRefund()
claimRefund(projectId: 0, amount: 220_000, merkleProof);
// REVERTS: Alice is a winner, not in refund merkle tree

// Option B: Look for claimExtraRefund()
claimExtraRefund(projectId: 0, poolId: 0);
// FUNCTION DOESN'T EXIST

```

**Result:** Alice paid 220,000 USDT and received tokens worth that amount at pool price, but she expected to be refunded and get the tokens for free. She loses 100% of her expected refund.
is is not an exploitable vulnerability but a missing feature:

**Step 1:** Owner creates Pool 0 with extra refund enabled
```javascript
createPool(projectId: 0, totalAllocation: 2_200_000, tokenPrice: 0.10, hasExtraRefund: true);
// Pool 0 created, hasExtraRefund flag set to true
```

**Step 2:** Alice reads whitepaper and sees the promise
```
"Pool 1 Discount = 80% + Full USDT refund = More tokens & for free"
```

**Step 3:** Alice bids 220,000 USDT expecting to get it refunded
```javascript
placeBid(projectId: 0, amount: 220_000 USDT, vestingPeriod: 365 days);
// Alice commits 220,000 USDT
```

**Step 4:** Bidding ends, Alice wins allocation in Pool 0

**Step 5:** Alice claims her NFT
```javascript
claimNFT(projectId: 0, poolId: 0, amount: 2_200_000, merkleProof);
// Alice receives NFT with 2,200,000 A26Z tokens vesting
```

**Step 6:** Alice tries to claim her refund
```javascript
// Option A: Try claimRefund()
claimRefund(projectId: 0, amount: 220_000, merkleProof);
// REVERTS: Alice is a winner, not in refund merkle tree

// Option B: Look for claimExtraRefund()
claimExtraRefund(projectId: 0, poolId: 0);
// FUNCTION DOESN'T EXIST

```

**Result:** Alice paid 220,000 USDT and received tokens worth that amount at pool price, but she expected to be refunded and get the tokens for free. She loses 100% of her expected refund.


### Impact

- Users who bid for Pool 0 or last pool based on whitepaper promises receive no refund
- Actual outcome: Full payment with no refund
- Loss per affected user: 100% of their bid amount
- Affects all winners in Pool 0 and last pool of every IWO project


### PoC

_No response_

### Mitigation

Add the missing `claimExtraRefund()` function so winner of first and last pool can get refunes as expected in Whitepaper


  