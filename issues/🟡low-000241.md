# [000241] Lack of Event Emission in `AlignerzVesting::withdrawStuckETH()` Reduces Transparency
  
  - **Description**

The `::withdrawStuckETH()` allows the woner to withdraw `ETH` from the protocol, but no event is emitted to record this action on-chain.

- **Impact**

`ETH` withdrawals are invisible to event-reliant monitoring systems.

- **Recommended Mitigation**

Add an event declaration and emit it in the function:

```diff
// Add to events section
+ event StuckETHWithdrawn(address indexed to, uint256 amount);

// Update function
function withdrawStuckETH(uint256 amount) external onlyOwner {
    require(amount > 0, Zero_Value());
    require(address(this).balance >= amount, Insufficient_Balance());

    (bool success,) = msg.sender.call{value: amount}("");
    require(success, Transfer_Failed());
+   emit StuckETHWithdrawn(msg.sender, amount);
}
```
  