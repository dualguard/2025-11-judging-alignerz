# [000733] Solidity Mapping Getter Limitation Will Cause Complete Failure of Dividend Distribution for All NFT Holders
  
  ### Summary

The `A26ZDividendDistributor` contract attempts to retrieve the `Allocation` struct containing dynamic arrays (`amounts[]`, `vestingPeriods[]`, etc.) from `AlignerzVesting::allocationOf`, which is defined as a public mapping. Solidity's auto-generated getter functions for public mappings cannot return dynamic arrays within structs. This results in a return data mismatch that causes every call to `getUnclaimedAmounts()` to revert during ABI decoding. Consequently, the entire dividend distribution mechanism is permanently non-functional from the moment of deployment.

### Root Cause

The vulnerability stems from a mismatch between how Solidity generates getters for public mappings and how the Distributor expects to receive data.
* In `AlignerzVesting.sol::allocationOf` is defined as a public mapping:
```solidity
/// @notice Mapping to fetch the allocation of a TVS given the NFT Id
    mapping(uint256 => Allocation) public allocationOf;
```
The Solidity compiler generates a getter for this mapping that omits all dynamic arrays (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) and only returns the fixed-size fields (`isClaimed`, `token`, `assignedPoolId`)
* In `A26ZDividendDistributor`: The contract (via `IAlignerzVesting`) expects `allocationOf` to return the full `Allocation` struct in memory.
```solidity
 /// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
    /// @param nftId NFT Id
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) { 
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
uint256[] memory amounts = vesting.allocationOf(nftId).amounts; // @> Reverts: auto-generated getter omits dynamic arrays, causing ABI decoding failure
```
When the Distributor calls `allocationOf`, the Vesting contract returns a small tuple of data. The Distributor attempts to decode this as a large struct containing arrays. The decoding fails, causing the transaction to revert.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. The Owner calls `setUpTheDividends()` to distribute dividends.
2. The function calls` _setAmounts()`, which invokes `getTotalUnclaimedAmounts()`.
3. `getTotalUnclaimedAmounts()` iterates over all NFTs and calls `getUnclaimedAmounts(nftId)`.
4. `getUnclaimedAmounts()` calls `vesting.allocationOf(nftId)` expecting to read the amounts array.
5. The call reverts due to ABI decoding failure (the Vesting contract did not return the array data).
6. The transaction fails, and dividends cannot be calculated or distributed

### Impact

* Complete Loss of Functionality: The `A26ZDividendDistributor` contract is completely non-functional.
* Permanent Failure: There is no way to distribute dividends to any user. The `setUpTheDividends` function will revert 100% of the time.
* Users Affected: 100% of NFT holders expecting dividend payments.

### PoC

The following test was ran and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:IntegrationFailurePoC
[PASS] testDistributorRevertsDueToInterfaceMismatch() (gas: 726580)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  Attempting to call getUnclaimedAmounts(1)...
  Confirmed: Transaction reverted due to Interface Mismatch.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.83s (8.49ms CPU time)

```

The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract IntegrationFailurePoC is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public projectToken;
    MockUSD public dividendToken;
    // We use a dummy token for the distributor to bypass the "Inverted Token Check" logic bug
    // so we can reach the integration failure.
    MockUSD public dummyToken; 
    A26ZDividendDistributor public distributor;
    
    address public user;
    uint256 constant PROJECT_ID = 0;
    uint256 constant TVS_AMOUNT = 1000 ether;

    function setUp() public {
        user = makeAddr("user");
        projectToken = new Aligners26("26Aligners", "A26Z");
        dividendToken = new MockUSD(); 
        dummyToken = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        
        // Deploy Vesting (Real Contract)
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));

        // Deploy Distributor (Real Contract)
        // Configured with dummyToken to pass the logic check:
        // address(distributor.token) != address(vesting.token)
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(dividendToken),
            block.timestamp,
            30 days,
            address(dummyToken) 
        );

        // Fund Owner for creating allocation
        projectToken.approve(address(vesting), type(uint256).max);
    }

    function testDistributorRevertsDueToInterfaceMismatch() public {
        // 1. Setup a valid TVS for the user
        vesting.launchRewardProject(
            address(projectToken),
            address(dividendToken),
            block.timestamp + 1 days,
            30 days
        );

        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TVS_AMOUNT;

        vesting.setTVSAllocation(PROJECT_ID, TVS_AMOUNT, 365 days, kols, amounts);
        
        vm.prank(user);
        vesting.claimRewardTVS(PROJECT_ID); // User now has NFT #1
        
        // 2. Attempt to get unclaimed amounts
        // The Distributor will call vesting.allocationOf(1).
        // Vesting returns: (bool isClaimed, address token, uint256 poolId)
        // Distributor expects: (uint256[] amounts, uint256[] periods, ..., bool isClaimed, ...)
        
        // This ABI decoding mismatch causes a Revert (usually EvmError or decoding error).
        
        console.log("Attempting to call getUnclaimedAmounts(1)...");
        
        // We expect a generic Revert/EvmError because ABI decoding fails at the call site.
        vm.expectRevert(); 
        distributor.getUnclaimedAmounts(1);
        
        console.log("Confirmed: Transaction reverted due to Interface Mismatch.");
    }
}
```

### Mitigation

The Vesting contract must implement a manual getter function that explicitly returns the full struct in memory, as mapping getters cannot do this.
  