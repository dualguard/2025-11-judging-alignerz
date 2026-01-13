# [000516] Rounding-Induced `require()` Permanently Locks Vested Tokens Due to Integer Division
  
  ### Summary

When `claimableAmount` rounds down to `0` due to integer division (even though `claimableSeconds > 0`), the `require(claimableAmount > 0)` causes the entire claim to revert. Since `claimedSeconds` is never updated, every future claim fails the same way, permanently locking tokens.

### Root Cause


 Integer division rounds down → `claimableAmount = 0` when `amount * claimableSeconds < vestingPeriod` and `claimableSeconds` is not 0. The `require()` treats this as an error and reverts. But `claimableSeconds > 0` (time HAS passed). Since the revert happens before `claimedSeconds` is updated, leads to progress is never recorded. 


 In `AlignerzVesting.sol` in `getClaimableAmountAndSeconds()` function -

```solidity
    getClaimableAmountAndSeconds(...){

     //...rest code 
       claimableSeconds = secondsPassed - claimedSeconds;
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
 @>     require(claimableAmount > 0, No_Claimable_Tokens());
        return (claimableAmount, claimableSeconds);
    }
```
So, when `amount * claimableSeconds < vestingPeriod`  and `claimableSeconds` is not 0 on that case function will always revert. Leads to User not able to claim his rest token if exist on other flow index (those flow index will be execute after this flow index and flow index is `allocation.vestingPeriods.length`).

Code Snippet -
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996


### Internal Pre-conditions

1. Small residual vesting of users where `amount * claimableSeconds < vestingPeriod` and `claimableSeconds` is not 0.
2.  User has multiple allocations with indices `[0, 1, 2, 3, 4]` and `claimableAmount` will be  0 due to rounding happen on starting indices (e.g., 0 or 1).


### External Pre-conditions

N/A

### Attack Path


Assuming User has multiple allocations with indices `[0, 1, 2, 3, 4]`  and when user call `claimTokens()` then users 0 indices `allocation amount` is 100 and `vestingPeriod` is 10,000  and `claimableSeconds` is 10 and  `allocation amount` for remaining indices `1,2,3,4` is `100,100,100,100`.

1. User calls `claimTokens(projectId, nftId)`
   
2. Loop reaches flow index `i=0` where claimableAmount rounds to 0.
   
3. then `getClaimableAmountAndSeconds()` executes with :
   - claimableSeconds = 10 
   - claimableAmount = (100 * 10) / 10,000 = 0
   - require(claimableAmount > 0) --> function will REVERT
   
4. Transaction reverts BEFORE this line executes:
   allocation.claimedSeconds[i] += claimableSeconds;
   
5. State unchanged → claimedSeconds[i] still = old value.
   
6. Next claim(calling `claimTokens(projectId, nftId)`) repeats steps 1-5 infinitely.
   
7. Tokens permanently locked + NFT cannot burn.


### Impact

Permanent lock of remaining vested tokens. NFT cannot be burned, allocation cannot be fully claimed.

### PoC

N/A

### Mitigation

Remove the require from claimable amount.
  