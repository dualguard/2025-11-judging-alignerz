# [001083] Amounts and Dividends cant be set in A26ZDividendDistributor because of Unbounded For Loop
  
  ### Summary

The function `A26ZDividendDistributor::getUnclaimedAmounts` contains a logic error in its main loop.
Inside the loop, two conditions use continue:
```solidity
if (claimedFlows[i]) continue;

if (claimedSeconds[i] == 0) {
    amount += amounts[i];
    continue;
}
```
However, neither branch increments `i`, and the `i++` only occurs at the bottom of the loop.
This means that whenever either condition is true, the loop returns to the top without increasing i, causing it to **get stuck on the same index forever**.
As a result, the loop becomes unbounded, leading to an infinite loop and eventually **reverting with out-of-gas**.

Any NFT that has either a fully claimed vesting flow `(claimedFlows[i] == true)` or a flow where claiming has not started `(claimedSeconds[i] == 0)` will cause the function to revert.

Since `getUnclaimedAmounts` is used inside `getTotalUnclaimedAmounts`, `_setAmounts`, and ultimately `setUpTheDividends` and `setAmounts`, this bug prevents dividends from ever being initialized.

### Root Cause

The issue is caused by an unbounded for loop that results in an Out of Gas revert.

In the lines below , the loop uses continue without incrementing i:

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L151

These branches run when a vesting flow is fully claimed `(claimedFlows[i] == true)` or not claimed at all `(claimedSeconds[i] == 0)`.
When either condition is true, the loop jumps back to the top **without increasing i, keeping the same index forever**.
This creates an infinite loop, causing the function to run **out of gas** and revert for any NFT that meets these conditions.

### Internal Pre-conditions

1. The owner must transfer the dividend amount to the deployed `A26ZDividendDistributor` contract.
2. Owner must to call `A26ZDividendDistributor::setUpTheDividends` or `A26ZDividendDistributor::setAmounts` 

### External Pre-conditions

1. AlignerzVesting must be deployed, and at least one project must be created.
2. Users must submit bids for that project, the project owner must finalize all bids, and atleast one user must claim their NFTs.
3. The project must announce dividends and deploy an instance of `A26ZDividendDistributor` with the correct parameters for that specific project.
4. Among the project’s token holders, at least one user must have either **fully claimed** their tokens or **not claimed any tokens** at all.


### Attack Path

1. An attacker (or any user) holds a project NFT where the vesting flow is either fully unclaimed `(claimedSeconds[i] == 0)` or fully claimed `(claimedFlows[i] == true)`.
2. The owner calls `setUpTheDividends` or `setAmounts` on `A26ZDividendDistributor`.
3. Inside these functions, `getTotalUnclaimedAmounts()` is executed, which calls `getUnclaimedAmounts()` for each NFT.
4. When processing the attacker’s NFT, the loop in `getUnclaimedAmounts()` hits a continue statement without incrementing i, causing an **infinite loop**.
5. The function runs out of gas and **reverts**, making dividend setup impossible.

### Impact

Dividend distribution cannot be set up because the contract always reverts because of **incorrect logic in for loop**. This makes **dividend distribution impossible**, and any **tokens sent to the distributor get stuck until the owner manually withdraws them**. The team must redeploy a fixed distributor contract for dividends to work.

### PoC

_No response_

### Mitigation

```diff
contract A26ZDividendDistributor{
    .
    .
    .
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
-           if (claimedFlows[i]) continue;
+           if (claimedFlows[i]){
+               unchecked {
+                   ++i;
+               }
+               continue;
+           }
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
-               continue;
+               unchecked {
+                   ++i;
+               }
+               continue;
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
   .
   .
}
  