# [000421] setTotalUnclaimedAmounts(), _setDividends(), and setUpTheDividends() Can Be Bricked by State Growth (Gas DoS)
  
  The functions `getTotalUnclaimedAmounts()`, `_setDividends()`, and indirectly `setUpTheDividends()` iterate over **all minted NFTs** in the collection without any gas limit protection or batching mechanism.

As the NFT collection grows, these functions will eventually exceed the **block gas limit**, rendering them **impossible to execute**. This creates a permanent denial-of-service (DoS) condition in which dividends can **never be set up or recalculated**, effectively locking any funds allocated for distribution.

---

## Proof of Concept (Foundry Test)

The following Foundry test demonstrates the issue:

```solidity
function testDOS() public {
    // Mint NFT to Alice
    uint256 NFTNum = 1;
    for (uint256 i = 0; i < NFTNum; i++) {
        nft.mint(alice);
        stablecoin.mint(address(alice), 1000e6); // 1000 USDC
    }

    uint256 nftCount = nft.getTotalMinted();
    console2.log("Total NFTs minted:", nftCount);

    vm.warp(startTime + 1 days);

    uint256 startGas = gasleft();
    vm.prank(owner);
    distributor.setUpTheDividends();
    uint256 endGas = gasleft();
    uint256 gasUsed = startGas - endGas;

    console2.log("Gas Used", gasUsed, "for NFT", nftCount);
}
```

* Gas used for **1 NFT**: ~39,695 gas
* Typical Ethereum block gas limit: ~30,000,000 gas (30M)
* Estimated NFTs needed to reach block limit:

```NFTs ≈ 30,000,000 / 39,695 ≈ 756```


> This demonstrates that **roughly 756 NFTs or more** will cause `setUpTheDividends()` to exceed the block gas limit, making dividend processing impossible. In real scenarios, due to storage writes and events, DoS may occur even sooner, depending on the number of unique owners.

---

## Impact

* **Permanent contract failure**: Once the NFT collection grows beyond the gas-processable limit (~1000–3000 NFTs depending on logic complexity), dividend distribution becomes impossible.
* **Funds locked**: Dividends cannot be recalculated or claimed, effectively locking the funds permanently.
* **Potential for gas-based attacks**: Malicious actors could intentionally mint many NFTs or manipulate state to trigger a DoS.

---

## Root Cause

1. **Unbounded iteration**: The functions iterate over all NFTs without batching.
2. **No gas check or limit**: There is no mechanism to break the loop or process in chunks.
3. **Dependence on dynamic state**: `totalUnclaimedAmounts` calculation depends on NFT allocations; if allocations are zero or large, the gas cost scales linearly with the number of NFTs.

---

## Mitigation Strategies

1. **Batch Processing / Pagination**

   * Split dividend calculations into **smaller chunks**, e.g., 50–100 NFTs per transaction.
   * Store progress in contract state, allowing multiple calls to process all NFTs incrementally.

2. **Limit NFT Count Per Transaction**

   * Enforce a maximum number of NFTs processed per dividend claim or setup to ensure transactions stay below block gas limits.

3. **Off-chain Calculation with On-chain Merkle Proofs**

   * Calculate dividend allocations off-chain.
   * Use **Merkle trees** to verify entitlements on-chain without iterating over all NFTs.

4. **Guard Against Zero Divisors**

   * Ensure that `totalUnclaimedAmounts` is **never zero** before performing division.
   * Add a check to skip processing or revert safely if no unclaimed amounts exist.

5. **Gas-Optimized Storage**

   * Minimize storage writes and events inside loops.
   * Aggregate dividend allocations for the same owner to reduce repeated SSTORE costs.

---

## Conclusion

The current implementation of dividend distribution in `A26ZDividendDistributor` is **vulnerable to a state growth gas DoS**. Without batching or off-chain calculation, the system cannot scale safely beyond a few hundred NFTs.

Implementing **batch processing, off-chain calculations, and safe divisor checks** is essential to prevent permanent DoS and ensure reliable dividend distribution for large NFT collections.

  