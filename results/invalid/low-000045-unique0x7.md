# [000045] Owner can withdraw  Reward Tokens or TVS Tokens via `withdrawStuckTokens` Function
  
  **Note: `A26ZDividendDistributor::withdrawStuckTokens()` also has the same issue**


##  Summary
The `withdrawStuckTokens` function in the `AlignerzVesting` contract allows the owner to withdraw any ERC20 tokens held by the contract. The function is implemented as follows:

```solidity
function withdrawStuckTokens(address tokenAddress, uint256 amount) external onlyOwner {
    require(amount > 0, Zero_Value());
    require(tokenAddress != address(0), Zero_Address());

    IERC20 token = IERC20(tokenAddress);
    require(token.balanceOf(address(this)) >= amount, Insufficient_Balance());
    token.safeTransfer(msg.sender, amount);
}
```
which doesn't check if the token is part of an active project.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L374

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L178

## Impact

- The function does not differentiate between "stuck" tokens and tokens actively used in the contract (e.g., reward tokens or TVS tokens) this means the owner can withdraw  any ERC20 token held by the contract, including tokens allocated for rewards or vesting.


## Recommendation
To prevent the withdrawal of reward tokens or TVS tokens, the `withdrawStuckTokens` function should include a check to exclude tokens actively used in the contract. 


  