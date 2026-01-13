# [001084] Project token Owners Can Steal Dividends of Different different projects
  
  ### Summary

The function `A26ZDividendDistributor::getUnclaimedAmounts` contains an incorrect token-matching check when determining whether an NFT belongs to the same project as the dividend distributor.
The function currently performs:
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
However, this condition is reversed.
The expected behavior is: If the **project token** associated with dividend **does not** match the token associated with the NFT’s allocation, then the NFT belongs to a different project and should receive **zero dividends**.

Instead, the current logic returns **0** when the **tokens match**, which is the opposite of what is intended.
Because of this inversion, valid NFTs are incorrectly skipped from dividend calculations, while NFTs belonging to other projects are processed incorrectly. This leads to faulty dividend distribution and breaks the correctness of the reward logic across different projects.

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141

In the above line, the token-matching check is implemented incorrectly.
The line:
```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
is meant to skip NFTs that belong to a different project by returning 0 when the tokens differ.
Instead, the condition is reversed: it returns 0 when the tokens match, and processes NFTs from other projects.

This inverted condition causes **valid NFTs** to be **skipped** and incorrectly includes NFTs from **unrelated projects** in the unclaimed-amount calculation.

### Internal Pre-conditions

1. The owner must transfer the dividend amount to the deployed `A26ZDividendDistributor` contract.
2. Owner must to call `A26ZDividendDistributor::setUpTheDividends`

### External Pre-conditions

1. AlignerzVesting must be deployed, and at least two projects must be created.
2. Users must submit bids for that respective projects, the project owners must finalize all bids, and atleast one user from different projects must claim their NFTs.
3. One of the project must announce dividends and deploy an instance of `A26ZDividendDistributor` with the correct parameters for that specific project.

### Attack Path

1. Attacker acquires an NFT from **Project A**  by bidding and claiming the NFT. The attacker does not need to claim any Project A tokens.
2. **Project B** Owner calls setUpTheDividends() or setAmounts() to initialize a new dividend distribution for Project B holders.
3. Dividend distribution logic internally calls `A26ZDividendDistributor::getUnclaimedAmounts()` to fetch each NFT’s pending dividend amount.
4. Due to a project-matching bug, the function incorrectly returns 0 for NFTs belonging to the same project **(Project B)** and returns a positive unclaimed amount for NFTs belonging to a different project **(Project A)**.
5. Attacker then calls the dividend claim function, and because their NFT comes from **Project A**, the buggy logic treats it as eligible for **Project B’s** dividends and assigns them a non-zero amount.

### Impact

The legitimate token holders of the **dividend-issuing project suffer a loss of all dividends distributed**, because their NFTs are incorrectly treated as ineligible and **receive 0 payout**.
The attacker gains a part of the entire dividend amount allocated to that project, as NFTs from other projects are incorrectly treated as eligible and can claim all unclaimed dividend tokens.

### PoC

_No response_

### Mitigation

```diff
contract A26ZDividendDistributor{
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-       if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+       if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
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
}
```
  