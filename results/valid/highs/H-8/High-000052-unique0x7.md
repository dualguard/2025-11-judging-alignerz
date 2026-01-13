# [000052] Infinite loop in `getUnclaimedAmounts()`
  
  ### Summary

The function `A26ZDividendDistributor::getUnclaimedAmounts()` iterates through vesting flow arrays using:

```solidity
for (uint i; i < len;) {
    if (claimedFlows[i]) continue;
    ...
    unchecked { ++i; }
}
```

The issue is that `continue` jumps to the next loop iteration **without executing `++i`**, because the increment is placed **inside the last block** of the loop body.

When `claimedFlows[i]` is `true`, the following happens:

1. `continue` is hit.
    
2. The loop restarts.
    
3. `i` is **not incremented**.
    
4. Execution repeats forever with the same `i`.
    

This results in an **infinite loop**, which eventually **consumes all gas and reverts**.



### Root Cause

```solidity

    /// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
    /// @param nftId NFT Id
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;       //<<<<<@,
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140


### Impact

- Infinite loop, which eventually  consumes all gas and reverts.
-  Denial-of-Service vulnerability for core functionality.

### PoC

### Create a file named `A26ZDividendDistributorTest.t.sol` and add the following code.
 run `forge test --mt test__infiniteLoopIngetUnclaimedAmounts -vvvv`
````solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";


contract A26ZDividendDistributorTest is Test {

     AlignerzVesting public vesting;
    A26ZDividendDistributor public dividendDistributor;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);

        dividendDistributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), 1731811200, 7776000, address(token));
    }
    
    function test__infiniteLoopIngetUnclaimedAmounts() public {
        // forge's vm.mockCall lets us stub vesting.allocationOf to return crafted data
        uint256 nftId = 0;

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1e18;

        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 123;

        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 1 days;

        uint256[] memory vestingStartTimes = new uint256[](1);
        vestingStartTimes[0] = block.timestamp;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = true; // triggers the buggy continue path

        IAlignerzVesting.Allocation memory allocation = IAlignerzVesting.Allocation({
            amounts: amounts,
            vestingPeriods: vestingPeriods,
            vestingStartTimes: vestingStartTimes,
            claimedSeconds: claimedSeconds,
            claimedFlows: claimedFlows,
            isClaimed: false,
            token: IERC20(address(0xBEEF)), // different from dividendDistributor.token()
            assignedPoolId: 0
        });

        vm.mockCall(
            address(vesting),
            abi.encodeWithSelector(IAlignerzVesting.allocationOf.selector, nftId),
            abi.encode(allocation)
        );

        vm.expectRevert();
        dividendDistributor.getUnclaimedAmounts(nftId);
    }



}

````
**Result**
```solidity
├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [1056919831] A26ZDividendDistributor::getUnclaimedAmounts(0)
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   ├─ [0] ERC1967Proxy::fallback(0) [staticcall]
    │   │   └─ ← [Return] true, 0x0000000000000000000000000000000000000100, 320
    │   └─ ← [OutOfGas] EvmError: OutOfGas
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.60s (6.48s CPU time)
````


### or simply paste this into Remix IDE

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";


contract Owner {


    function goodLoop() public pure returns (uint256 total) {
        for (uint i = 0; i < 5; i++) 
        {
            if (i == 2) {
                continue ;
            }
            total += i; 
        }
        return total;
    }



      function badLoop() public pure returns (uint256 total) {
        for (uint i = 0; i < 5;) 
        {
            if (i == 2) {
                continue ;
            }
            total += i; 

            unchecked {
            i++;
        }
        }
        
        return total;
    }
} 

````

### Mitigation

```diff
 /// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
    /// @param nftId NFT Id
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) 
+            {
+                unchecked { ++i; }
                continue;
+            }
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
+                unchecked { ++i; }
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```
  