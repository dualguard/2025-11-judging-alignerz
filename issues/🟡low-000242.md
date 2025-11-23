# [000242] Lack of Event Emission in `A26ZDividendDistributor::withdrawStuckTokens()` Reduces Transparency
  
  - **Description**

The `::withdrawStuckTokens()` allows the owner to withdraw any `ERC20` tokens from the contract, but no event is emitted to record this action on-chain.

- **Impact**

While this function is protected by an `onlyOwner` modifier, the lack of event emission means there is no permanent on-chain record of which tokens were withdrawn, when, and in what amounts. This is particularly important for monitoring and analytical tools.

- **Recommended Mitigation**

Add an event declaration and emit it in the function:

```diff
// Add to events section
+ event StuckTokensWithdrawn(address indexed token, address indexed to, uint256 amount);

// Update function
function withdrawStuckTokens(address tokenAddress, uint256 amount) external onlyOwner {
    require(amount > 0, Zero_Value());
    require(tokenAddress != address(0), Zero_Address());

    IERC20 token = IERC20(tokenAddress);
    require(token.balanceOf(address(this)) >= amount, Insufficient_Balance());
    token.safeTransfer(msg.sender, amount);
+   emit StuckTokensWithdrawn(tokenAddress, msg.sender, amount);
}
```
  