# [000060] Front-Running finalizeBids() Allows Last-Second Sniping of Allocations with Perfect Information
  
  ## Summary

Users can front-run the [`finalizeBids()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L783) transaction to place last-second bids after the admin has decided to close the auction. The `placeBid()` function checks `block.timestamp <= biddingProject.endTime` before the state change, allowing attackers to see the finalization transaction in the mempool and submit bids that will be included in the allocation despite the admin's intent to close bidding.

## Vulnerability Details

The `placeBid()` function validates bidding is active using the current state:

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    // ...
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(
        block.timestamp >= biddingProject.startTime &&
        block.timestamp <= biddingProject.endTime &&  // ← Checks endTime
        !biddingProject.closed,                        // ← Checks closed status
        Bidding_Period_Is_Not_Active()
    );
    // ... accept the bid
}
```

When `finalizeBids()` is called, it updates these values:

```solidity
function finalizeBids(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots, uint256 claimWindow)
    external
    onlyOwner
{
    // ...
    biddingProject.closed = true;
    biddingProject.endTime = block.timestamp;  // ← Updates endTime to "now"
    // ...
}
```

### The Attack Flow

1. Admin launches project with `endTime = block.timestamp + 7 days`
2. Days 1-3: Honest users place bids (all visible on-chain via `BidPlaced` events)
3. After 3 days, admin decides to close early and calls `finalizeBids()`
4. Attacker monitors mempool and sees the `finalizeBids()` transaction
5. Attacker analyzes all previous bids from on-chain data:
   - Sees which users bid how much and with what vesting periods
   - Calculates optimal bid parameters to maximize their allocation
   - Knows exactly when bidding will close (in the next block)
6. Attacker front-runs with `placeBid()`:
   - Check: `block.timestamp` (day 3) `<= endTime` (day 7) passes
   - Check: `!closed`passes
   - Bid is accepted with optimized parameters
7. `finalizeBids()` executes in the same block
8. Attacker's bid is included in the allocation despite being placed after the admin decided to close

### Why This Works

The vulnerability exists because:

- `placeBid()` reads state that hasn't been updated yet
- Both transactions can execute in the same block
- The checks use the "old" values before `finalizeBids()` modifies them
- There's no protection against same-block bidding during finalization

## Impact

Attackers can gain unfair advantages by:

1. Perfect information advantage: By the time `finalizeBids()` is called, all previous bids are visible on-chain through `BidPlaced` events. The attacker can analyze:

   - Total bid amounts per user
   - Vesting periods chosen by each bidder
   - The complete competitive landscape

2. Optimal bid calculation: With full knowledge of all competing bids, the attacker can calculate the exact bid amount and vesting period needed to maximize their allocation or secure a spot in the best pool.

3. Last-mover advantage: Honest bidders made decisions without knowing the final state of competition. The front-runner gets to bid with complete information about all competitors.

4. Bypassing the intended close time: The admin's decision to close bidding is undermined by last-second front-running bids that shouldn't be possible.

This is particularly problematic because the whitepaper describes mechanisms to prevent "last-minute bid manipulation," but this front-running vector completely bypasses those protections.

  