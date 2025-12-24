# [000614] updateBid function doesn’t take updateBidFee into account & will fail in many cases
  
  ### Summary

`AlignerzVesting::updateBid` function takes the `uint256 newAmount`  as parameter, and after subtracting the oldAmount, transfers the additional amount from the user. But also, if `updateBidFee` is greater than 0, it attempts to transfer that additional `updateBidFee` from the user, which will fail if the user didn’t give enough allowance for the update fee on each function call.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L770

### Details

On `updateBid` function, the `newAmount` is given by the user in params. The function takes this amount from the user first, then attempts to take the `updateBidFee` (if it’s greater than 0) afterwards. 

The problem is, this fee is not reduced from the user given amount, but accounted separately. Consider the following scenario:

- User wants to update their bid from 5e18 tokens to 10e18 tokens.
- They give allowance for 5e18 tokens for security reasons, a it is not advised to give max allowance, and they want to spend exactly 5e18 tokens.
- The contract first sends the 5e18 tokens as follows:

additional amount will be calculated as 10e18 - 5e18= 5e18

```jsx
       if (newAmount > oldAmount) {
            uint256 additionalAmount = newAmount - oldAmount;
            biddingProject.totalStablecoinBalance += additionalAmount;
            biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), additionalAmount);
        }
```

- `updateBidFee` is greater than 0, say, 1e16. The contract attempts to transfer this amount to treasury as follows:

```jsx
   if (updateBidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, updateBidFee);
        }
```

- The above call will fail due to insufficient allowance, because the caller never gave allowance for that extra updateBidFee amount.

This function will revert each and every time the user gives the exact amount of allowance for the newAmount param they pass.

The updateBidFee can be updated by owner at any time, thus it cannot be expected from users to keep track of what the fee is at all times, and they cannot figure how much they should give allowance on each call, it will be too confusing for them. They will just get revert each time they try to bid.

### Impact

DOS of bidding update functionality. It cannot be assumed that the user will give extra, or max allowance.

### Recommendation

You can subtract the fee from the user specified amount, and let them know that there’s a fee and their bid will be slightly lower than they pass. 

In this case, if updateBidFee is 1e16 and the user wants to update their bid to 10e18 from 5e18 tokens, they should be informed that their bid will be incremented by 5e18 - 1e16
  