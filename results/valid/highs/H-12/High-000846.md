# [000846] Inverted Token Logic in Dividend Distribution Enables Cross-Project Fund Theft
  
    ## Summary

  The `getUnclaimedAmounts` function in `A26ZDividendDistributor.sol` contains critically inverted logic that enables holders of different project tokens to steal dividend rewards while legitimate token holders receive zero allocation, completely inverting the protocol's economic incentives across multi-project scenarios.

  ## Vulnerability Detail

  The dividend distribution system contains inverted conditional logic that processes wrong project tokens and ignores correct tokens, enabling economic attacks where holders of different project TVS
  positions can drain dividend pools intended for other projects:

  **Location**: [`A26ZDividendDistributor.sol:141`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141)

  ```solidity
  function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
      if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; // ❌ BUG: Inverted logic
      // Function only processes TVS with DIFFERENT tokens!

      uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
      uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
      uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
      bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
      uint256 len = vesting.allocationOf(nftId).amounts.length;

      for (uint i; i < len;) {
          // Calculates dividend allocation from wrong project token amounts!
          if (claimedFlows[i]) continue;
          if (claimedSeconds[i] == 0) {
              amount += amounts[i];
              continue;
          }
          uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
          uint256 unclaimedAmount = amounts[i] - claimedAmount;
          amount += unclaimedAmount;
          unchecked { ++i; }
      }
      unclaimedAmountsIn[nftId] = amount;
  }

  Root Cause: The condition checks if tokens match and returns 0 (wrong), but should return 0 when tokens DON'T match. This creates a complete economic inversion where:
  - ✅ Matching tokens (token == TVS.token) → Returns 0 allocation
  - ❌ Different tokens (token != TVS.token) → Calculates full allocation

  Multi-Project Context: The protocol supports multiple projects with different tokens simultaneously. Owner can create Project A with A26Z tokens and Project B with WETH tokens. When dividend distributors
  are created for specific projects, the inverted logic causes cross-project theft.

  Critical Impact Chain: This bug affects all dividend distribution through
  https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L131-L135 and
  https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224.

  Attack Mechanism

  1. Multi-Project Setup: Protocol has Project A (A26Z tokens) and Project B (WETH tokens)
  2. Legitimate Holdings: Alice holds A26Z TVS, Bob holds WETH TVS from different projects
  3. Dividend Announcement: A26Z dividend distributor announces 10,000 USDC distribution
  4. Economic Inversion: Due to inverted logic, Bob's WETH tokens get processed while Alice's A26Z tokens get 0
  5. Cross-Project Theft: Bob drains dividend pool intended for A26Z holders
  6. Fund Theft: Bob steals dividends from Project A while holding Project B tokens

  Step-by-Step Cross-Project Attack:

  // Setup: A26ZDividendDistributor distributes to Project A ($A26Z token holders)
  // Dividend Pool: 10,000 USDC to distribute to A26Z holders

  // Legitimate Project A Holder: Alice
  address(token) = $A26Z_ADDRESS                           // Distributor targets A26Z
  address(vesting.allocationOf(aliceNFT).token) = $A26Z_ADDRESS    // Alice's TVS contains A26Z
  // Logic: if ($A26Z == $A26Z) return 0;                  // ❌ TRUE → Alice gets 0 allocation

  // Project B Holder: Bob
  address(token) = $A26Z_ADDRESS                           // Distributor targets A26Z
  address(vesting.allocationOf(bobNFT).token) = $WETH_ADDRESS      // Bob's TVS contains WETH
  // Logic: if ($A26Z == $WETH) return 0;                  // ❌ FALSE → Bob gets full allocation

  // Result: Bob steals Alice's Project A dividends using Project B position!

  Cross-Function Impact:
  // Line 131-135: Total calculation includes wrong project tokens
  function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
      uint256 len = nft.getTotalMinted();
      for (uint i; i < len;) {
          (, bool isOwned) = safeOwnerOf(i);
          if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i); // Uses broken function
          // Alice (A26Z): +0, Bob (WETH): +3000 → Total = 3000 (wrong project!)
      }
  }

  // Line 214-224: Distribution uses corrupted totals
  function _setDividends() internal {
      uint256 len = nft.getTotalMinted();
      for (uint i; i < len;) {
          (address owner, bool isOwned) = safeOwnerOf(i);
          if (isOwned) {
              dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
              // Alice (A26Z): (0 * 10000 / 3000) = 0 USDC ❌
              // Bob (WETH): (3000 * 10000 / 3000) = 10000 USDC ❌ (steals Project A dividends!)
          }
      }
  }

  Impact

  High Severity: Complete economic inversion enabling cross-project fund theft

  - Cross-Project Fund Theft: Holders of different project tokens can drain dividend pools from other projects
  - Legitimate Holder Loss: Valid project token holders receive zero dividends despite being entitled to rewards
  - Economic Incentive Inversion: Protocol rewards wrong project participation and punishes correct project participation
  - Multi-Project Economic Failure: Dividend distributions completely broken across project boundaries
  - Scalable Attack: Attackers can repeat across multiple projects and dividend distributions

  Economic Attack Scenarios:

  // Scenario 1: Cross-Project Theft
  Project A Dividend Pool: 10,000 USDC for A26Z holders
  - Alice (5000 A26Z TVS): Gets 0 USDC ❌ (loses entitled dividends)
  - Bob (100 WETH TVS from Project B): Gets 10,000 USDC ❌ (steals Project A pool)

  // Scenario 2: Multiple Project Attack
  Project A Dividend Pool: 50,000 USDC for A26Z holders
  - Legitimate A26Z holders (100,000 A26Z): Get 0 USDC ❌
  - Other project token holders (mixed projects): Get 50,000 USDC ❌
  - Result: Project A participants get nothing, other projects benefit

  Code Snippet

  Current vulnerable implementation:
  function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
      // ❌ CRITICAL BUG: Inverted conditional logic
      if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
      //                 ^^ This should be != (not equal)

      // This calculation only runs for DIFFERENT project tokens!
      uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
      uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
      uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
      bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
      uint256 len = vesting.allocationOf(nftId).amounts.length;

      for (uint i; i < len;) {
          if (claimedFlows[i]) continue;
          if (claimedSeconds[i] == 0) {
              amount += amounts[i]; // Adds wrong project token amounts!
              continue;
          }
          uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
          uint256 unclaimedAmount = amounts[i] - claimedAmount;
          amount += unclaimedAmount; // More wrong project amounts!
          unchecked { ++i; }
      }
      unclaimedAmountsIn[nftId] = amount; // Stores wrong allocation
  }

  Tool used

  Manual Review

  Recommendation

  Fix the inverted conditional logic:

  function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
      // ✅ FIXED: Correct conditional logic
      if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
      //                 ^^ Changed == to !=

      // Now this calculation only runs for MATCHING project tokens
      uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
      uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
      uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
      bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
      uint256 len = vesting.allocationOf(nftId).amounts.length;

      for (uint i; i < len;) {
          if (claimedFlows[i]) continue;
          if (claimedSeconds[i] == 0) {
              amount += amounts[i]; // Now adds correct project token amounts
              continue;
          }
          uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
          uint256 unclaimedAmount = amounts[i] - claimedAmount;
          amount += unclaimedAmount;
          unchecked { ++i; }
      }
      unclaimedAmountsIn[nftId] = amount;
  }

  This ensures dividend distributors only calculate allocations for TVS positions containing the intended project tokens, preventing cross-project fund theft.
  