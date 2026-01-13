# [000615] placeBid function doesn’t take bidFee into account & will fail in many cases
  
  ### Summary

`AlignerzVesting::placeBid` function takes the `uint256 amount` as parameter, and transfers that amount from the user. But also, if `bidFee` is greater than 0, it attempts to transfer that additional `bidFee` from the user, which will fail if the user didn’t give enough allowance for the fee on each function call.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L728

### Details

On `placeBid` function, the amount is given by the user in params. The function takes this amount from the user first, then attempts to take the `bidFee` (if it’s greater than 0) afterwards. 

The problem is, this fee is not reduced from the user given amount, but accounted separately. Consider the following scenario:

- User wants to bid 10e18 tokens.
- They give allowance for 10e18 tokens for security reasons, a it is not advised to give max allowance, and they want to spend exactly 10e18 tokens.
- The contract first sends the 10e18 tokens as follows:

```jsx
        biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);

```

- bidFee is greater than 0, say, 1e16. The contract attempts to transfer this amount to treasury as follows:

```jsx
   if (bidFee > 0) {
            biddingProject.stablecoin.safeTransferFrom(msg.sender, treasury, bidFee);
        }
```

- The above call will fail due to insufficient allowance, because the caller never gave allowance for that extra bidFee amount.

This function will revert each and every time the user gives the exact amount of allowance for the amount param they pass.

The bidFee can be updated by owner at any time, thus it cannot be expected from users to keep track of what the fee is at all times, and they cannot figure how much they should give allowance on each call, it will be too confusing for them. They will just get revert each time they try to bid.

### Impact

DOS of bidding functionality. It cannot be assumed that the user will give extra, or max allowance.

### Recommendation

You can subtract the fee from the user specified amount, and let them know that there’s a fee and their bid will be slightly lower than they pass. 

In this case, if bidFee is 1e16 and the user wants to bid 10e18 tokens, they should be informed that their bid will be 10e18 - 1e16
  