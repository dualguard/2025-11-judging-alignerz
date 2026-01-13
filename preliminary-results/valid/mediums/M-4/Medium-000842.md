# [000842] ERC721A Token ID 1 Start Causes Off-By-One Error Leading to Last NFT Dividend Exclusion
  
  ## Summary
The ERC721A implementation starts token IDs at 1 instead of 0, but the dividend distribution system incorrectly assumes 0-based indexing, causing a systematic off-by-one error that permanently excludes the last minted NFT from all dividend calculations and distributions.

## Vulnerability Detail
The AlignerzNFT contract uses ERC721A with `_startTokenId() = 1`, creating token IDs 1, 2, 3, etc. However, the dividend distribution functions use 0-based iteration patterns that systematically exclude the highest token ID from dividend calculations.

### Critical Issue: Dividend Calculation Off-By-One Error
```solidity
// A26ZDividendDistributor.sol:127-135 - BROKEN ITERATION LOGIC
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted(); // Returns COUNT of tokens (e.g., 5)
    for (uint i; i < len;) {            // Loops: i = 0, 1, 2, 3, 4
        (, bool isOwned) = safeOwnerOf(i); // Queries tokens 0, 1, 2, 3, 4
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
    // ❌ BUG: Token IDs are 1, 2, 3, 4, 5 but only processes 0, 1, 2, 3, 4
    // ❌ RESULT: Token 5 (last NFT) is NEVER processed for dividends
}
```

### Critical Issue: Dividend Distribution Exclusion
```solidity
// A26ZDividendDistributor.sol:214-224 - SAME BUG IN DISTRIBUTION
function _setDividends() internal {
    uint256 len = nft.getTotalMinted(); // Count = 5 tokens
    for (uint i; i < len;) {            // Loops: i = 0, 1, 2, 3, 4
        (address owner, bool isOwned) = safeOwnerOf(i); // Queries tokens 0, 1, 2, 3, 4
        if (isOwned) dividendsOf[owner].amount += 
            (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked { ++i; }
    }
    emit dividendsSet();
    // ❌ BUG: Last minted NFT (token 5) receives ZERO dividends
}
```

### Token ID Mismatch Analysis
```solidity
// ERC721A.sol:105-107 - Token IDs start at 1
function _startTokenId() internal pure virtual returns (uint256) {
    return 1; // ❌ This creates the fundamental mismatch
}

// When 5 NFTs are minted:
// - nft.getTotalMinted() returns 5
// - Actual token IDs: 1, 2, 3, 4, 5
// - Loop processes: i = 0, 1, 2, 3, 4
// - Token 0: safeOwnerOf(0) returns (address(0), false) - safely ignored
// - Tokens 1-4: Correctly processed
// - Token 5: COMPLETELY MISSED - never processed
```

## Impact

### Direct Financial Loss
Owner of highest token ID receives nothing
No mechanism to recover missed dividends
Happens every dividend distribution

`getTotalUnclaimedAmounts()` underestimates by last token's value
All other holders get slightly inflated dividends
Creates unfair advantage for early minters vs late minters

### Economic Attack Vector
```solidity
// Attack Scenario:
// 1. Attacker observes dividend distribution timing
// 2. Right before distribution, attacker mints final NFT
// 3. Attacker's NFT becomes "last minted" → excluded from dividends
// 4. Attacker can grief protocol by causing systematic fund loss
```

## Attack Mechanism

### Systematic Exclusion Attack
1. **Setup Phase**: Monitor dividend distribution contract for upcoming `setUpTheDividends()` calls
2. **Timing Attack**: Mint NFT just before dividend calculation to become "last token"
3. **Dividend Loss**: Last token systematically excluded from all dividend calculations
4. **Repeat**: Every dividend round excludes the current last NFT holder

### Proof of Concept Scenario
```
Initial State:
- 3 NFTs minted: Token IDs 1, 2, 3
- getTotalMinted() = 3
- Dividend iteration: i = 0, 1, 2
- Processes: Token 0 (none), Token 1 ✓, Token 2 ✓
- MISSES: Token 3 ❌

After 4th NFT mint:
- 4 NFTs minted: Token IDs 1, 2, 3, 4  
- getTotalMinted() = 4
- Dividend iteration: i = 0, 1, 2, 3
- Processes: Token 0 (none), Token 1 ✓, Token 2 ✓, Token 3 ✓
- MISSES: Token 4 ❌
```

## Code Snippet

### Vulnerable Iteration Pattern
```solidity
// A26ZDividendDistributor.sol:127-135 - getTotalUnclaimedAmounts
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted(); // ❌ Returns COUNT, not max ID
    for (uint i; i < len;) {            // ❌ 0-based iteration
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked { ++i; }
    }
}

// A26ZDividendDistributor.sol:214-224 - _setDividends  
function _setDividends() internal {
    uint256 len = nft.getTotalMinted(); // ❌ Same bug
    for (uint i; i < len;) {            // ❌ Same pattern
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += 
            (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked { ++i; }
    }
    emit dividendsSet();
}
```

### Safe Ownership Check Function
```solidity
// A26ZDividendDistributor.sol:164-173 - safeOwnerOf correctly handles non-existent tokens
function safeOwnerOf(uint256 nftId) public view returns (address owner, bool exists) {
    try nft.extOwnerOf(nftId) returns (address _owner) {
        return (_owner, true);  // ✓ Returns actual owner
    } catch {
        return (address(0), false); // ✓ Handles token 0 and burned tokens gracefully
    }
}
```

## POC

### Demonstrating the Off-By-One Error
```solidity
function testDividendExclusionOffByOne() public {
    vm.startPrank(owner);
    
    // Mint 3 NFTs - Token IDs will be 1, 2, 3
    uint256 token1 = nft.mint(user1); // ID = 1
    uint256 token2 = nft.mint(user2); // ID = 2  
    uint256 token3 = nft.mint(user3); // ID = 3
    
    console.log("Token IDs:", token1, token2, token3);
    console.log("Total minted:", nft.getTotalMinted()); // = 3
    
    // Setup dividend distribution
    usdc.mint(address(dividendDistributor), 1000e6);
    dividendDistributor.setUpTheDividends();
    
    // Check dividend allocations
    (uint256 user1Dividends,) = dividendDistributor.dividendsOf(user1);
    (uint256 user2Dividends,) = dividendDistributor.dividendsOf(user2); 
    (uint256 user3Dividends,) = dividendDistributor.dividendsOf(user3);
    
    console.log("User1 (Token 1) dividends:", user1Dividends); // ✓ Receives dividends
    console.log("User2 (Token 2) dividends:", user2Dividends); // ✓ Receives dividends
    console.log("User3 (Token 3) dividends:", user3Dividends); // ❌ ZERO dividends!
    
    // Token 3 holder gets NO dividends despite having valid NFT
    assertEq(user3Dividends, 0, "Last NFT excluded from dividends");
    
    vm.stopPrank();
}
```


This vulnerability causes direct fund loss for the last NFT holder in every dividend distribution.
  