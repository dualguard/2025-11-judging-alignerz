# [000020] High: `getUnclaimedAmount()` hits an infinite loop
  
  ### Summary

Normal allocations are created with `claimedSeconds = 0`. Any later attempt to compute unclaimed amounts for such NFTs via `getUnclaimedAmounts` will hit an infinite loop pattern (OOG) due to `continue` skipping the manual `++i`, causing a permanent DoS of all flows that depend on computing `unclaimedAmounts`. 

### Root Cause

In [`getUnclaimedAmount()`](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161), the `for` header has an empty increment section:
```solidity
for (uint i; i < len; ) { ... }
                  ^^^
          // nothing here, so the loop itself does *not* increment i
```
Next, the code increment's `i` manually in the body. 
```solidity
unchecked { ++i; }
```

In a `for (init; conditional; post)`, `continue` jumps to the `post` part and then runs the conditional. But here, `post` is empty, so `continue` jumps back to rechecking `i < len` without running any code in between. This means that anytime we hit `contine`, we skip the manual `++i`.

To buttress, suppose: 
- `len = 3`
- At `i = 0`, `claimedFlows[0] == true`

Execution:
- Start: `i = 0
- Check `i < len` ->`0 < 3` -> true -> enter body of the code
- `if (claimedFlows[0]) continue;` -> this is true -> execute `continue`

Effect:
- `continue` jumps to the top of the loop
- It does not run `++i` because `++i` is in the body, after the continue
- There is no post clause in the for header to increment `i`
- Back at top: `i` is still 0
- Check `i < len` -> `0 < 3` -> still true
- Evaluate body again, hit `claimedFlows[0]` again, continue again and so on

You are now stuck at `i = 0` forever. Same story if `claimedSeconds[i] == 0` is true: you hit that continue, skip ++i, and never move on. This leads to an OOG revert. 


### Internal Pre-conditions

1. There is no post clause in the for header to increment `i`
2. `continue` is hit before manual `++i` is implemented

### External Pre-conditions

Nil

### Attack Path

1. Protocol launches, users earn or are granted allocations. Those allocations are stored with `claimedSeconds = 0 and claimedFlows = false` by design
2. Protocol later tries to distribute new dividends, or
3. As part of that, it calls `getUnclaimedAmounts(nftId)` 
4. The first NFT with a fresh/unclaimed tranche triggers the infinite loop path.
5. The entire transaction runs out of gas and reverts.

### Impact

High impact
The bug causes `getUnclaimedAmounts(nftId)` to run out of gas / revert whenever there is at least one tranche with `claimedSeconds[i] == 0` or `claimedFlows[i] `== true. In normal usage, freshly created allocations have `claimedSeconds[i] == 0` and `claimedFlows[i] == false`, so any call over a fresh allocation is already broken. This means that reward/dividend accounting cannot be updated on-chain and any logic that relies on the affected function becomes unusable. This is a severe disruption of protocol functionality

Likelihood is High as the issue triggers in normal dividend flows.

### PoC


### Mitigation
Refactor the loop to use a normal for increment and remove manual ++i

```diff
 function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; 
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
-       for (uint i; i < len;) {  
+      for (uint i; i < len; ++i) { 
            if (claimedFlows[i]) continue; //@audit note we dont move on to the next after continue. I is not incremented, instead we loop forever on this same i
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
-           unchecked {
-                 ++i;
-             } 
        }
        unclaimedAmountsIn[nftId] = amount; 
    }
```

  