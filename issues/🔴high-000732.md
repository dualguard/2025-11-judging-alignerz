# [000732] Inverted Token Check Disables Rewards for Valid Vesting Schedules
  
  ### Summary

The `A26ZDividendDistributor::getUnclaimedAmounts` function contains a logic error that prevents users from receiving rewards if they hold the correct vesting token. The condition checks if the token in the distributor matches the token in the vesting schedule and returns 0 if they match. This effectively ensures that only invalid or unrelated vesting schedules (which likely don't exist) would be eligible for calculation, while valid holders get nothing.

### Root Cause

This logic is inverted. The intention was likely `!=` to ensure the vesting schedule matches the reward token before proceeding. Instead, it forces a return of 0 whenever the tokens match. 
```solidity
/// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
    /// @param nftId NFT Id
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) { 
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0; //@> Inverted check, anyone who holds the rewarded token gets 0 rewards
```

### Internal Pre-conditions

1. The `A26ZDividendDistributor` is deployed with a specific reward token (e.g.,`A26Z`).
2. A user holds a Vesting NFT that vests the same token (`A26Z`).
3. The Critical Integration Failure (ABI decoding crash) must be fixed to reach this line of code.

### External Pre-conditions

None

### Attack Path

1.  The admin configures the Distributor to reward holders of the project token.
2. The admin calls `setUpTheDividends`.
3. The contract iterates through user NFTs. For each valid NFT, it checks: `if (DistributorToken == VestingToken)`.
4. Since the tokens match (as they should), the condition is true, and the function returns 0.
5. No dividends are calculated for any legitimate user. The total distributed amount remains 0.

### Impact

The dividend distribution mechanism is functionally broken,  this logic ensures that 0 dividends are distributed to all valid users.

### PoC

The following test was ran and produced these logs:
```bash
Ran 1 test for test/PoC.t.sol:InvertedTokenCheckPoC
[PASS] testRewardsAreZeroWhenTokenMatches() (gas: 1321095)
Logs:
  Token in Distributor: 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Token in Vesting:     0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  Unclaimed Amount:     0
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

// 1. Mock Vesting Contract
// This Mock returns the full struct properly, bypassing the "Critical Integration Failure".
// It allows us to isolate and test the Logic Bug.
contract MockVesting is IAlignerzVesting {
    address public vestingToken;

    constructor(address _token) {
        vestingToken = _token;
    }

    function allocationOf(uint256) external view returns (IAlignerzVesting.Allocation memory alloc) {
        alloc.token = IERC20(vestingToken); // <--- Crucial: Sets the token to match/mismatch the Distributor
        
        // Populate arrays with valid data so calculation logic works if reached
        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = 1000 ether; 
        
        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 100 days;

        alloc.vestingStartTimes = new uint256[](1);
        // Avoid underflow when running in test environments where block.timestamp may be small.
        if (block.timestamp >= 50 days) {
            alloc.vestingStartTimes[0] = block.timestamp - 50 days; // Partial vesting
        } else {
            alloc.vestingStartTimes[0] = 0;
        }

        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 0;
        alloc.claimedFlows = new bool[](1);
        alloc.claimedFlows[0] = false;
        
        return alloc;
    }
   

    // Required Interface Stubs
    function claimRewardTVS(uint256) external {}
    function claimTokens(uint256, uint256) external {}
    // ... add other interface functions if strictly needed by compilation, usually mocks are loose in Foundry
 }

// 2. Mock NFT to satisfy constructor
contract MockNFT is IAlignerzNFT {
    function extOwnerOf(uint256) external pure returns (address) { return address(0x1); }
    function getTotalMinted() external pure returns (uint256) { return 1; }
    function mint(address) external returns (uint256) { return 0; }
    function burn(uint256) external {}
    function addMinter(address) external {}
}

contract InvertedTokenCheckPoC is Test {
    A26ZDividendDistributor public distributor;
    MockVesting public vesting;
    MockNFT public nft;
    MockUSD public targetToken;
    MockUSD public stablecoin;

    function setUp() public {
        targetToken = new MockUSD(); // The token we WANT to reward (e.g., A26Z)
        stablecoin = new MockUSD();
        nft = new MockNFT();
    }

    function testRewardsAreZeroWhenTokenMatches() public {
        // Scenario: The Distributor is set up to reward holders of 'targetToken'.
        // The Vesting contract confirms the user holds 'targetToken'.
        // BUG EXPECTATION: The distributor returns 0 because of the inverted check (==).

        // 1. Deploy Vesting that holds the CORRECT token
        vesting = new MockVesting(address(targetToken));

        // 2. Deploy Distributor configured for that SAME token
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(targetToken) // <--- Matches Vesting
        );

        // 3. Call getUnclaimedAmounts
        uint256 unclaimed = distributor.getUnclaimedAmounts(1);

        console.log("Token in Distributor:", address(targetToken));
        console.log("Token in Vesting:    ", vesting.vestingToken()); // MockVesting public variable
        console.log("Unclaimed Amount:    ", unclaimed);

        // 4. Assert Failure
        // If the logic was correct (token == token), we should get a calculation result (e.g. 1000).
        // Because of the bug, we get 0.
        assertEq(unclaimed, 0, "Bug Confirmed: Returns 0 even though tokens match");
    }
    
}
```

### Mitigation

Invert the check to `!=`.
```diff
- if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  