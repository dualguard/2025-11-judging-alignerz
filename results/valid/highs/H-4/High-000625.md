# [000625] Precision Loss Causes DoS for All NFT Claims When One Flow Has Dust Remainder
  
  ### Summary

Solidity truncation in `getClaimableAmountAndSeconds` will cause a complete DoS of token claiming for users who have any NFT flow with dust remainder near vesting completion, as the function reverts when claimableAmount = 0, and since claimTokens iterates through all flows without skip logic, one reverting flow blocks claiming from all other flows across all NFTs.

### Root Cause

In [AlignerzVesting.sol:getClaimableAmountAndSeconds](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980C5-L996C6) the function calculates claimable amounts using integer division which truncates:
```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;
require(claimableAmount > 0, No_Claimable_Tokens());
```
When `claimableSeconds * amount ` is very small, the division result can round down to 0. The function then reverts with `No_Claimable_Tokens()`.
This becomes critical because in [AlignerzVesting.sol:claimTokens](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L958), the function loops through all flows without try-catch or skip logic:
```solidity
for (uint256 i; i < nbOfFlows; i++) {
    if (allocation.claimedFlows[i]) {
        flowsClaimed++;
        continue;  //  Can skip fully claimed flows
    }
    (uint256 claimableAmount, uint256 claimableSeconds) = 
        getClaimableAmountAndSeconds(allocation, i);  //  Reverts if claimableAmount = 0
    // ...
}
```
If any flow has claimableAmount = 0, the entire transaction reverts, preventing the user from claiming any flow from any NFT.


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

VestingPeriod = 31556926 seconds (365 days)
// Last time claimed, near to vesting end period
claimedSeconds = 31556826
vestingStartTime = 1735686000 //1jan2025
// due to multiple split
amount = 100 wei
block.timestamp now = 1769900400 //1feb2025
```solidity
        // 1769900400 > 31556926  +  1735686000 = 1767242926 
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            // secondsPassed  = 31556926 
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }
        // 31556926  - 31556826 = 100
        claimableSeconds = secondsPassed - claimedSeconds;
        // 100 * 100 / 31556926  = 0 => due to solidity truncation
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
        require(claimableAmount > 0, No_Claimable_Tokens());
```
### Impact
User cannot claim from any NFT, if one flow result claimableAmount = 0

### PoC

_No response_

### Mitigation

Try to get claimable amount, skip if zero
  