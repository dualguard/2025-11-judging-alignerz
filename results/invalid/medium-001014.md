# [001014] The "refund + keep allocation" incentive can be gamed
  
  ### Summary


Users can game the "refund + keep allocation" incentive system by placing bids with extremely high vesting periods (e.g. `type(uint256).max`), guaranteeing placement in Pool 1 while receiving both a full refund and token allocation.

### Root Cause

According to the whitepaper section `4.1.2. Bidding Window: Eliminating Manipulation`:
```text
We also decided to add additional incentives such as first pool & last pool winners will keep
their allocation & get refunded. 
```
The protocol's allocation prioritizes bids with longer vesting periods for Pool 1, which offers:
- The lowest token price
- Additional incentive: bidders keep their allocation AND receive a full refund (when `hasExtraRefund = true`)

Bidders who knows that Pool 1 has `hasExtraRefund = true` can:
1. Place bids with `vestingPeriod = type(uint256).max` (or any extremely large value)
2. Guarantee placement in Pool 1 due to longest vesting period
3. Receive a full refund of their stablecoin bid
4. Still retain the token allocation (vesting forever but claimable in small amounts)

The protocol assumes users will balance vesting duration against their need for liquidity. However, when refunds are guaranteed, this assumption breaks down. There is no economic disincentive to choosing an infinite vesting period.


### Internal Pre-conditions

1. First pool to refund the bidders; `createPool` is called with `hasExtraRefund = true`.

### External Pre-conditions

None

### Attack Path

- Admin create 1st pool with additional incentives; `hasExtraRefund = true` 
- Alice places a 10,000 USDT bid with `vestingPeriod = type(uint256).max`
- Alice is allocated to Pool 1 (lowest price, eg. $0.10 per token)
- Alice receives 100,000 tokens + gets her 10,000 USDT refunded
- While the tokens vest over a very long timeframe, Alice has zero financial risk since she was refunded

### Impact

-  Pool 1 fills with bids that have no real economic commitment, defeating the "diamond hands" incentive mechanism.
- Users who genuinely want longer vesting periods with reasonable timeframes are crowded out by manipulated bids

### PoC

_No response_

### Mitigation

Consider implementing one of the below changes:
- Refund just a fraction of amount paid.
- Full refund just a % of users (eg. 30%) included in the pool; Since users don't know if they will get refunded or not, they are disincentivized to lock their capital for very long period of time by setting the vesting period to uint256.max.

  