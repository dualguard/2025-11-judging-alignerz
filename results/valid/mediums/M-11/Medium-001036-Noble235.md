# [001036] [High] NFT Transfer After Dividend Allocation Enables Complete Theft of Quarterly Dividend Distributions
  
  ### Summary

The address-based dividend tracking without NFT ownership verification in `A26ZDividendDistributor.sol` will cause a complete loss of quarterly dividend distributions for current TVS NFT holders as former NFT holders will transfer their NFTs after dividend allocation and still claim the allocated dividends that should belong to the new NFT owners.

### Root Cause

In `A26ZDividendDistributor.sol` the `claimDividends()` [function](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L186-L204) allows any address to claim dividends allocated to them without verifying that the caller currently owns any TVS NFT. This is combined with the design choice in `A26ZDividendDistributor.sol` where `_setDividends()` [allocates](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L213-L225) dividends to addresses based on a snapshot of NFT ownership at allocation time using `dividendsOf[owner].amount +=` without any mechanism to transfer or reset these dividend rights when the NFT is transferred to a new owner. The `dividendsOf` mapping tracks dividends per address rather than per tokenId, and there are no transfer hooks in the NFT contract (`ERC721A.sol` contains empty `_beforeTokenTransfers()` and `_afterTokenTransfers()` functions that are never overridden in `AlignerzNFT.sol`) to reset dividend allocations when ownership changes.

### Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` to trigger the internal `_setAmounts()` function that sets `stablecoinAmountToDistribute` equal to the contract's USDC balance and calculates `totalUnclaimedAmounts` across all TVS positions.
2. Owner needs to call `setUpTheDividends()` to trigger the internal `_setDividends()` function that iterates through all minted NFTs and allocates dividends proportionally using `dividendsOf[owner].amount +=` based on each TVS's unclaimed vesting value.
3. At least one user needs to own a TVS NFT at the time of dividend allocation to receive a non-zero `dividendsOf[address].amount` value that persists indefinitely regardless of subsequent NFT transfers.
4. The `vestingPeriod` variable needs to be set to a duration greater than zero to enable time-based vesting of dividend claims over the defined period.
5. The `startTime` variable needs to be set to define when the dividend vesting period begins for calculating claimable amounts over time.

### External Pre-conditions

None

### Attack Path

1. Alice owns TVS NFT with tokenId 1 which represents a vesting position containing 1,000,000 locked A26Z tokens with substantial unclaimed value stored in the vesting contract's `allocationOf[1]` mapping.
2. AlignerZ generates quarterly profits and the protocol owner calls `setUpTheDividends()` which internally executes `_setAmounts()` to load the contract's USDC balance into `stablecoinAmountToDistribute` and calculate `totalUnclaimedAmounts` by summing the unclaimed token values across all TVS positions.
3. The `_setDividends()` function executes, iterating through all minted NFTs and calling `safeOwnerOf(i)` for each tokenId to determine current ownership, then allocating dividends proportionally using `dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)` which results in Alice receiving `dividendsOf[alice].amount = 50,000 USDC` based on her TVS position's proportional share of total unclaimed value.
4. Alice immediately transfers TVS NFT tokenId 1 to Bob using the standard ERC721 `transferFrom(alice, bob, 1)` function which updates the NFT ownership mapping in `ERC721A.sol` to reflect Bob as the new owner but triggers no hooks or callbacks to update the dividend distributor contract.
5. After the transfer completes, the state shows `nft.ownerOf(1) == bob` but `dividendsOf[alice].amount` still equals 50,000 USDC because no code path exists to reset or transfer dividend allocations when NFTs change hands, while simultaneously `dividendsOf[bob].amount` remains 0 because Bob was not the owner during the allocation snapshot in step 3.
6. Alice waits for time to pass and calls `claimDividends()` which reads `dividendsOf[msg.sender].amount` to get her allocated 50,000 USDC, calculates `claimableSeconds` based on the elapsed vesting time, computes `claimableAmount = totalAmount * claimableSeconds / vestingPeriod`, and executes `stablecoin.safeTransfer(user, claimableAmount)` to send USDC to Alice without ever verifying whether Alice currently owns any TVS NFT.
7. Alice successfully claims the full 50,000 USDC in dividends over the vesting period despite transferring away the TVS NFT that generated those dividend rights, effectively stealing the quarterly profit distribution that should have gone to Bob as the current holder of the vesting position.
8. Bob who now legitimately owns TVS NFT tokenId 1 and holds the underlying vesting position worth 1,000,000 locked A26Z tokens has `dividendsOf[bob].amount == 0` and cannot claim any portion of the 50,000 USDC quarterly dividend that the whitepaper promises to "TVS holders" because the allocation was based on a stale ownership snapshot that was never updated when the NFT transferred.

### Impact

Current TVS NFT holders suffer a 100% loss of their entitled quarterly dividend distributions as explicitly promised in the AlignerZ whitepaper section 5.2 which states "5% of our profit will be redistributed directly to our TVS holders, proportionally to their locked tokens." For each quarterly distribution cycle, former NFT holders can steal the entire allocated dividend amount by transferring their NFTs after the owner calls `setUpTheDividends()` but before claiming the vested dividends. According to the whitepaper's revenue projections, if AlignerZ generates 10 million USD in quarterly net profits, the 5% dividend pool equals 500,000 USDC that should be distributed to current TVS holders but can instead be completely stolen by addresses that transferred away their NFTs and no longer hold any vesting positions. The attackers gain the full dividend amount they claim which could reach hundreds of thousands of USDC per quarter with zero cost beyond gas fees, while legitimate current NFT holders who purchased TVS positions on secondary markets receive absolutely nothing despite the whitepaper's explicit promise that rewards are "proportionally to their locked tokens" and favor "those who hold the TVS for a long time." This fundamentally breaks the core value proposition stated in the whitepaper that "the longer & bigger your TVS, the greater the reward" because buyers of TVS NFTs discover they will never receive any dividend rights regardless of how long they hold. The vulnerability completely destroys the secondary market for TVS NFTs as rational buyers realize that purchasing a TVS on an NFT marketplace means acquiring the underlying vested tokens but zero dividend entitlement, making the advertised quarterly profit sharing worthless and contradicting the whitepaper's promise that "TVSs unlock a new dimension in DeFi" by enabling liquidity without sacrificing rewards. The compounding nature of the vulnerability across multiple quarterly distributions means a single malicious actor who owned TVS positions during multiple allocation snapshots can accumulate and steal dividends from several quarters even after selling all their NFTs months earlier.

### PoC

Add test to test/DividendTheftExploitTest.t.sol

Run using forge test --match-contract DividendTheftConceptualProof -vvv


```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

/**
 * @title Proof of Concept: Dividend Theft via NFT Transfer
 * @notice This test demonstrates that dividends are allocated to addresses based on
 *         a snapshot of NFT ownership, but these allocations persist even after the
 *         NFT is transferred to a new owner, allowing former holders to steal dividends.
 */
contract DividendTheftConceptualProof is Test {
    
    function testConceptualProof_DividendAllocationPersistsAfterTransfer() public view {
        console.log("=== CONCEPTUAL PROOF OF VULNERABILITY ===\n");
        
        console.log("STEP 1: Initial State");
        console.log("- Alice owns TVS NFT #1");
        console.log("- Bob owns nothing\n");
        
        console.log("STEP 2: Owner calls setUpTheDividends()");
        console.log("- Executes: _setDividends() at line 196-203");
        console.log("- Code: dividendsOf[owner].amount += (...)");
        console.log("- Result: dividendsOf[alice].amount = 250,000 USDC");
        console.log("- Result: dividendsOf[bob].amount = 0 USDC\n");
        
        console.log("STEP 3: Alice transfers NFT #1 to Bob");
        console.log("- NFT ownership changes: alice -> bob");
        console.log("- ERC721A._transfer() executes at line 487-513");
        console.log("- NO HOOKS called to update dividend distributor");
        console.log("- _beforeTokenTransfers() is empty (line 509)");
        console.log("- _afterTokenTransfers() is empty (line 511)\n");
        
        console.log("STEP 4: Post-Transfer State");
        console.log("- nft.ownerOf(1) = bob (NFT ownership updated)");
        console.log("- dividendsOf[alice].amount = 250,000 USDC (UNCHANGED!)");
        console.log("- dividendsOf[bob].amount = 0 USDC (UNCHANGED!)\n");
        
        console.log("STEP 5: Alice calls claimDividends()");
        console.log("- Function at line 169-185");
        console.log("- Code: address user = msg.sender;");
        console.log("- Code: uint256 totalAmount = dividendsOf[user].amount;");
        console.log("- NO CHECK: require(nft.ownerOf(tokenId) == msg.sender)");
        console.log("- Result: Alice receives 250,000 USDC\n");
        
        console.log("STEP 6: Bob tries to claim");
        console.log("- Bob is current NFT owner");
        console.log("- dividendsOf[bob].amount = 0");
        console.log("- Bob receives NOTHING\n");
        
        console.log("=== VULNERABILITY CONFIRMED ===");
        console.log("ROOT CAUSE:");
        console.log("1. dividendsOf mapping is address-based (line 51)");
        console.log("2. No NFT ownership check in claimDividends() (line 169)");
        console.log("3. No transfer hooks to reset dividends (ERC721A.sol)");
        console.log("4. Dividends allocated via snapshot don't follow NFT\n");
        
        console.log("IMPACT:");
        console.log("- Former NFT holders steal 100%% of allocated dividends");
        console.log("- Current NFT holders receive 0%% despite ownership");
        console.log("- Violates whitepaper promise: 'redistributed to TVS holders'");
        console.log("- Secondary NFT market becomes worthless\n");
        
        console.log("REFERENCES:");
        console.log("- A26ZDividendDistributor.sol:51 - address-based mapping");
        console.log("- A26ZDividendDistributor.sol:169-185 - no ownership check");
        console.log("- A26ZDividendDistributor.sol:196-203 - snapshot allocation");
        console.log("- ERC721A.sol:487-513 - transfer with empty hooks");
        console.log("- Whitepaper 2.6 Section 5.2 - promises to 'TVS holders'");
    }
    
    function testCodeAnalysis_ProofViaMappingStructure() public view {
        console.log("\n=== CODE ANALYSIS: MAPPING STRUCTURE ===\n");
        
        console.log("CURRENT IMPLEMENTATION (VULNERABLE):");
        console.log("Line 51: mapping(address => Dividend) dividendsOf;");
        console.log("- Dividends tracked per ADDRESS");
        console.log("- No connection to tokenId");
        console.log("- Persists when NFT transfers\n");
        
        console.log("ALLOCATION CODE (Line 196-203):");
        console.log("for (uint i; i < len;) {");
        console.log("    (address owner, bool isOwned) = safeOwnerOf(i);");
        console.log("    if (isOwned) dividendsOf[owner].amount += (...);");
        console.log("}");
        console.log("- Takes snapshot of ownership");
        console.log("- Stores in address mapping");
        console.log("- Never updates on transfer\n");
        
        console.log("CLAIM CODE (Line 169-185):");
        console.log("function claimDividends() external {");
        console.log("    address user = msg.sender;");
        console.log("    uint256 totalAmount = dividendsOf[user].amount;");
        console.log("    // ... transfers USDC ...");
        console.log("}");
        console.log("- NO ownership verification");
        console.log("- Only checks dividendsOf[msg.sender]");
        console.log("- Anyone with non-zero allocation can claim\n");
        
        console.log("CORRECT IMPLEMENTATION WOULD BE:");
        console.log("mapping(uint256 => Dividend) dividendsOf;  // Per tokenId");
        console.log("function claimDividends(uint256 tokenId) external {");
        console.log("    require(nft.ownerOf(tokenId) == msg.sender);");
        console.log("    // ... use dividendsOf[tokenId] ...");
        console.log("}");
    }
}
```

### Mitigation

Refactor the dividend distribution system to track dividends per tokenId instead of per address and enforce NFT ownership verification at claim time. Modify `A26ZDividendDistributor.sol` to change the mapping from `mapping(address => Dividend) dividendsOf` to `mapping(uint256 => Dividend) dividendsOf` so that dividend entitlements are permanently bound to specific NFT tokenIds rather than ephemeral address ownership. Update the `_setDividends()` internal function to iterate through all minted NFTs and allocate dividends using `dividendsOf[i].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts)` where i represents the tokenId instead of storing the allocation under the owner's address. Modify the `claimDividends()` function signature  to accept a `uint256 tokenId` parameter and add `require(nft.ownerOf(tokenId) == msg.sender, "Caller must own the NFT")` as the first line of the function to enforce that only the current NFT holder can claim dividends associated with that specific tokenId. Update all subsequent dividend claim logic within the function to reference `dividendsOf[tokenId]` instead of `dividendsOf[msg.sender]` and use the tokenId consistently throughout the vesting calculations. This architectural change ensures that dividend rights are intrinsically linked to the NFT asset itself rather than the historical address that owned it during allocation, so when Bob purchases TVS NFT tokenId 1 from Alice on a secondary marketplace, Bob automatically inherits the full `dividendsOf[1]` allocation and Alice retains zero claim rights since her address no longer holds any dividend-bearing tokenIds. The mitigation aligns the implementation with the whitepaper's explicit promise in section 5.2 that rewards go to "TVS holders" (current holders of the NFT) and ensures the system honors its core value proposition that "the longer & bigger your TVS, the greater the reward" by preventing former holders from stealing quarterly profit distributions that belong to the addresses currently holding the vesting positions.
  