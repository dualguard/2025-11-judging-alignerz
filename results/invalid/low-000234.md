# [000234] Users are charged `updateBidFee` for no-op updates in `updateBid` function
  
  ### Summary

The `updateBid` function allows users to modify their existing bids during an active bidding period. If a user calls this function with the same values as their existing bid (`newAmount` and `newVestingPeriod` are identical to the old values), the function does not revert. Instead, it proceeds to charge the `updateBidFee`, resulting in a direct loss of funds for the user without any corresponding change to their bid state.

### Root Cause
in `AlignerzVesting.sol: 741 - 777`

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L741-L776

The root cause of this issue is a missing validation check. The function validates that `newAmount >= oldAmount` and `newVestingPeriod >= bid.vestingPeriod`, which correctly prevents users from downgrading their bids. However, it fails to check if the new parameters are strictly different from the old ones. This allows a transaction where `newAmount == oldAmount` and `newVestingPeriod == bid.vestingPeriod` to proceed, triggering the fee payment logic for what is effectively a no-op (no-operation) update.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A user, Alice, has an active bid in a project where updateBidFee is greater than zero.
2. Alice calls `updateBid(projectId, newAmount, newVestingPeriod)`, where `newAmount` is equal to her current bid amount and `newVestingPeriod` is equal to her current vesting period.
3. The function's initial require statements all pass, as the new values are greater than or equal to the old values.
4. The logic block for handling an increased bid amount (`if (newAmount > oldAmount)`) is skipped.
5. The function reaches the fee payment logic at line 954: `if (updateBidFee > 0)`. This condition is true.
6. The contract executes `biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, updateBidFee)`, transferring the fee from Alice to the treasury.
7. The function then updates `bid.amount` and `bid.vestingPeriod` with the same values they already held.
8. The transaction successfully completes, and Alice has lost the `updateBidFee` for no reason.

### Impact

The impact of this vulnerability is a direct, unnecessary loss of funds for the user. While it does not put protocol funds at risk, it creates a "foot-gun" scenario where users can accidentally burn money. The severity is Low as the loss is contained to the user, limited to the `updateBidFee`, and requires an explicit transaction from the user.

### PoC

_No response_

### Mitigation

_No response_
  