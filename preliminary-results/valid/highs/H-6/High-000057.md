# [000057] Unbounded NFT Splitting Causes Permanent DoS of Dividend Distribution System
  
  ## Summary

The [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127) and [`setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L122) functions in `A26ZDividendDistributor` loop through all minted NFTs without any batching mechanism. Since `splitTVS()` has no limit on the number of pieces an NFT can be split into, a malicious user can intentionally create thousands of NFTs by repeatedly splitting, causing these functions to exceed the block gas limit and permanently brick the dividend distribution system.

## Vulnerability Details

### The Unbounded Loops

The dividend distribution system requires looping through ALL minted NFTs:

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();  // ← Total NFT count
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);  // ← External call
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);  // ← Multiple storage reads
        unchecked {
            ++i;
        }
    }
}

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();  // ← Total NFT count
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
}
```

These functions are called by `setUpTheDividends()`:

```solidity
function setUpTheDividends() external onlyOwner {
    _setAmounts();      // ← Calls getTotalUnclaimedAmounts()
    _setDividends();    // ← Loops through all NFTs again
}
```

### No Limit on NFT Splitting

The `splitTVS()` function allows users to split their NFT into unlimited pieces:

```solidity
function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,  // ← No length limit!
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    // ...
    uint256 nbOfTVS = percentages.length;  // ← No max check

    for (uint256 i; i < nbOfTVS;) {
        uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);  // ← Mints new NFTs
        // ... allocate percentages
    }

    require(sumOfPercentages == BASIS_POINT, Percentages_Do_Not_Add_Up_To_One_Hundred());
    // Only requirement: percentages must sum to 100%
}
```

### The Attack

A malicious user can weaponize this to permanently DoS the dividend system:

Step 1: Initial Bid

```solidity
// User places a bid and receives 1 TVS NFT
placeBid(projectId, 1000 USDC, vestingPeriod);
// After allocation: User owns NFT #1
```

Step 2: Recursive Splitting

```solidity
// Split into 100 pieces (1% each)
uint256[] memory percentages = new uint256[](100);
for (uint i = 0; i < 100; i++) {
    percentages[i] = 100;  // 1% each (100 basis points)
}
splitTVS(projectId, percentages, nftId);
// Result: 100 NFTs created

// Repeat for each new NFT
// After 2 rounds: 100 * 100 = 10,000 NFTs
// ...
```

Step 3: DoS Achieved

```solidity
// Admin tries to distribute dividends
setUpTheDividends();
// Reverts: Out of gas
// Dividend system is permanently broken
```

## Impact

### Permanent DoS of Core Functionality

The dividend system has two phases:

Phase 1: Allocation (Admin)

```solidity
setUpTheDividends() {
    _setAmounts();      // Loops through ALL NFTs to calculate total
    _setDividends();    // Loops through ALL NFTs to allocate shares
}
```

Phase 2: Claiming (Users)

```solidity
claimDividends() {
    uint256 totalAmount = dividendsOf[msg.sender].amount;  // Read allocated amount
    // ... calculate vested portion and transfer
}
```

The DoS occurs at Phase 1 (Allocation):

Once the NFT count exceeds the gas limit threshold:

- `setUpTheDividends()` will always revert during the allocation loops
- `dividendsOf[user].amount` is never set for any user
- Users cannot claim dividends because they were never allocated
- The entire 5% quarterly profit distribution feature is permanently bricked

Why there's no recovery:

- No batching mechanism exists for the allocation phase
- NFTs cannot be unminted or burned to reduce the count
- The merge function doesn't reduce total NFT count (just burns some and keeps others)
- Once the threshold is crossed, the system is permanently broken

### Weaponizable Attack

A malicious actor can intentionally cause this:

1. Low cost: Only pays split fees (0.5% per split)
2. Exponential growth: Each split round multiplies NFT count by N
3. Permanent damage: NFTs cannot be unminted or merged back
4. Affects all users: Entire dividend system breaks for everyone


### Affects All Token Holders

The whitepaper promises "5% of our profit will be redistributed directly to our TVS holders" - but this attack makes that impossible. All $A26Z holders lose their dividend rewards permanently.

## Severity

### High

- Permanent DoS: Core functionality (dividend distribution) is permanently broken
- Weaponizable: Attacker can intentionally cause this with low cost
- No recovery: No batching mechanism or way to reduce NFT count
- Affects all users: Entire protocol's dividend system is bricked
- Breaks whitepaper promise: The 5% profit distribution feature becomes impossible

### Escalation from Lightchaser Gas-14

Lightchaser identified this as [Gas-14] "Unbounded loop" as a gas optimization issue. However, they did not identify:

1. The weaponizable attack vector via unlimited NFT splitting
2. The permanent DoS of core functionality (not just high gas costs)
3. The low attack cost vs. high impact (millions in dividends lost)

This escalates the finding from Gas (optimization) to High (permanent DoS via intentional attack).

## Recommended Mitigation

Option 1: Limit Split Count (Immediate Fix)

Add a maximum limit to the number of pieces an NFT can be split into:

```solidity
uint256 public constant MAX_SPLIT_PIECES = 10;

function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    require(percentages.length <= MAX_SPLIT_PIECES, "Too many split pieces");

    // ... rest of logic
}
```

  