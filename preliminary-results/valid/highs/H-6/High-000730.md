# [000730] Unbounded Loops in Dividend Distribution Cause Permanent Denial of Service at Scale
  
  ### Summary

The `setUpTheDividends` function performs nested iterations over the entire user base (all minted NFTs). As the number of users grows, the gas required to execute this function scales linearly and will inevitably exceed the blockchain's block gas limit. Once this threshold is reached, the `setUpTheDividends` function will revert with "Out of Gas" 100% of the time, trapping all dividend funds in the contract forever.

### Root Cause

The vulnerability exists in `A26ZDividendDistributor.sol` due to O(N) complexity in state-changing loops:

* `getTotalUnclaimedAmounts` iterates 0 to `nft.getTotalMinted()`.
* `_setDividends` also iterates 0 to `nft.getTotalMinted()`.
* Inside these loops, multiple external calls (`safeOwnerOf`, `getUnclaimedAmounts`, `vesting.allocationOf`) are made, reading and writing to storage.
* The combined cost of these operations results in an exceptionally high gas burden per user.

### Internal Pre-conditions

1. The protocol mints a sufficient number of NFTs (users) such that the loop iteration cost exceeds the block gas limit.
2. The Admin attempts to call `setUpTheDividends` to distribute rewards.

### External Pre-conditions

None

### Attack Path

1. The project launches and attracts users. Over time, the number of NFT holders grows.
2.  The admin deposits stablecoins and calls `setUpTheDividends()` to reward these users.
3. The transaction iterates through the user list. The total gas cost is the sum of the gas required for each user iteration.
4.  The transaction exceeds the block gas limit and reverts.
5. There is no function to process users in batches or skip indices. The contract is now permanently broken; no future dividends can ever be distributed.

### Impact

1. The core reward mechanism becomes unusable very early in the project's lifecycle.
2. Any stablecoins deposited into the distributor for future rewards cannot be distributed.

### PoC

The following test was run and produced these logs:
```bash
Ran 1 test for test/PoC.t.sol:DoSGasScalingPoC
[PASS] testGasConsumptionScalesLinearly() (gas: 4574508)
Logs:
  ---------------------------------------
  Users (N):          50
  Total Gas Used:     4535137
  Avg Gas Per User:   90702
  ---------------------------------------
  Est. Max Users before permanent DoS: 330
  VULNERABILITY CONFIRMED: Protocol cannot scale beyond ~ 330 users.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 110.28ms (108.75ms CPU time)

Ran 1 test suite in 124.98ms (110.28ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
// 1. Mock Vesting to bypass other bugs and return valid data
contract MockVestingForDoS is IAlignerzVesting {
    IERC20 public vestingToken;

    constructor(address _token) { vestingToken = IERC20(_token); }

    function allocationOf(uint256) external view returns (IAlignerzVesting.Allocation memory alloc) {
        alloc.token = vestingToken; 
        
        // Return 1 flow per user to check loop cost
        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = 1000 ether;
        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 100 days;
        alloc.vestingStartTimes = new uint256[](1);
        // Avoid underflow in test environments where block.timestamp may be small
        if (block.timestamp >= 10 days) {
            alloc.vestingStartTimes[0] = block.timestamp - 10 days;
        } else {
            alloc.vestingStartTimes[0] = 0;
        }
        
        // Ensure claimedSeconds > 0 to avoid the 'Infinite Loop' bug
        // which would crash the test before we measure gas.
        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 1 days; 
        
        alloc.claimedFlows = new bool[](1);
        return alloc;
    }

    // Stubs
    function claimRewardTVS(uint256) external {}
    function claimTokens(uint256, uint256) external {}
}


// 2. Mock NFT to simulate user count
contract MockNFTForDoS is IAlignerzNFT {
    uint256 public totalMinted;
    
    function setTotalMinted(uint256 _total) external { totalMinted = _total; }
    function getTotalMinted() external view returns (uint256) { return totalMinted; }
    
    function extOwnerOf(uint256 tokenId) external view returns (address) {
        if (tokenId >= totalMinted) revert("OwnerQueryForNonexistentToken");
        // Return a deterministic address based on ID to simulate unique users
        return address(uint160(tokenId + 1000)); 
    }
    
    function mint(address) external returns (uint256) { return 0; }
    function burn(uint256) external {}
    function addMinter(address) external {}
}

contract DoSGasScalingPoC is Test {
    A26ZDividendDistributor public distributor;
    MockVestingForDoS public vesting;
    MockNFTForDoS public nft;
    MockUSD public stablecoin;
    MockUSD public token;

    function setUp() public {
        stablecoin = new MockUSD();
        token = new MockUSD(); // The vesting token
        nft = new MockNFTForDoS();
        vesting = new MockVestingForDoS(address(token));

        // Use address(0) for distributor token to bypass 'Inverted Token Check' logic bug
        // We want execution to actually reach the loops!
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(0) 
        );
        
        stablecoin.mint(address(distributor), 1_000_000 ether);
    }

    function testGasConsumptionScalesLinearly() public {
        // We simulate 50 users.
        // In the real world, this could be 5,000 or 50,000.
        uint256 userCount = 50;
        nft.setTotalMinted(userCount);

        uint256 gasStart = gasleft();
        
        // This function iterates 0 -> 50 twice (once in setAmounts, once in setDividends)
        distributor.setUpTheDividends();
        
        uint256 gasUsed = gasStart - gasleft();

        console.log("---------------------------------------");
        console.log("Users (N):         ", userCount);
        console.log("Total Gas Used:    ", gasUsed);
        
        uint256 gasPerUser = gasUsed / userCount;
        console.log("Avg Gas Per User:  ", gasPerUser);
        console.log("---------------------------------------");
        
        // ETH Block Gas Limit is 30,000,000
        uint256 blockLimit = 30_000_000;
        uint256 maxUsers = blockLimit / gasPerUser;
        
        console.log("Est. Max Users before permanent DoS:", maxUsers);

        // Verification Logic:
        // If 50 users take > 1M gas (~20k gas/user), then 1500 users kills the protocol.
        // A robust protocol should handle 10k+ users easily.
        
        assertTrue(gasUsed > 500_000, "Gas usage unreasonably low (loop likely optimized away)");
        
        // If max users is less than 5000, we consider this a critical scalability failure
        if (maxUsers < 5000) {
            console.log("VULNERABILITY CONFIRMED: Protocol cannot scale beyond ~", maxUsers, "users.");
        } else {
             console.log("Protocol scales acceptably.");
        }
    }
}
```

### Mitigation

Switch to a Merkle Distributor pattern ("Pull" model):

* Calculate all dividends off-chain using the event logs from the Vesting contract.
* Construct a Merkle Tree of eligible rewards.
* Admin calls `setMerkleRoot(bytes32 root, uint256 totalRewards)`.
* Users call claim`(uint256 amount, bytes32[] proof)` to pull their own rewards.
* This reduces the admin's gas cost to O(1) and shifts the claim cost to users, ensuring infinite scalability.
  