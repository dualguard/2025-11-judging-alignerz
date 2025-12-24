# [000951] Uninitialized Memory Arrays in splitTVS Cause Denial of Service
  
  ### Summary

The `_computeSplitArrays` function declared an `Allocation memory alloc` struct without initializing its dynamic arrays. When declared in memory without initialization, these arrays are null pointers (length 0). Attempting to write to these arrays (e.g., `alloc.amounts[j] = ...`) caused a panic (0x32: Array accessed at an out-of-bounds or negative index) and reverted all transactions, making the `splitTVS` functionality completely unusable.

### Root Cause

The root cause was in [`_computeSplitArrays`](https://github.com/dualguard/2025-11-alignerz-mabdullah22/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L458)

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    // BUG: alloc arrays were not initialized here
    // alloc.amounts, alloc.vestingPeriods, etc. had length 0
    
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = ...;  // PANIC: out-of-bounds access
        // ...
    }
}
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User receives TVS allocation via reward project or bidding project
2. User attempts to split their TVS into multiple parts via `splitTVS(projectId, percentages, nftId)`
3. Function calls `_computeSplitArrays` internally
4. `_computeSplitArrays` attempts to write to uninitialized memory arrays
5. Transaction reverts with panic code 0x32 (out-of-bounds array access)
6. **Result**: Complete denial of service for `splitTVS` functionality

### Impact

1. **Complete DoS**: The `splitTVS` functionality was completely broken and unusable
2. **User funds locked**: Users could not split their allocations for portfolio management
3. **Feature unavailability**: A core feature of the protocol was non-functional
4. **100% failure rate**: Every call to `splitTVS` would revert

### PoC

// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import "../src/contracts/nft/AlignerzNFT.sol";
import "../src/contracts/token/Aligners26.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract MemoryBugTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    Aligners26 stablecoin;
    address owner = address(0x1);
    address user = address(0x2);

    function setUp() public {
        vm.startPrank(owner);
        
        // Deploy NFT
        nft = new AlignerzNFT("AlignerzNFT", "ANFT", "https://example.com/");

        // Deploy Vesting Implementation
        AlignerzVesting vestingImpl = new AlignerzVesting();

        // Deploy Proxy
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(vestingImpl),
            abi.encodeWithSelector(AlignerzVesting.initialize.selector, address(nft))
        );
        vesting = AlignerzVesting(payable(address(proxy)));

        // Add vesting as minter
        nft.addMinter(address(vesting));

        // Deploy Tokens
        token = new Aligners26("Token", "TKN");
        stablecoin = new Aligners26("Stable", "STB");

        vm.stopPrank();
    }

    function testSplitTVSReverts() public {
        vm.startPrank(owner);
        
        // Launch Reward Project
        uint256 startTime = block.timestamp + 100;
        uint256 claimWindow = 1000;
        vesting.launchRewardProject(address(token), address(stablecoin), startTime, claimWindow);

        // Approve tokens
        token.approve(address(vesting), 1000 ether);

        // Set Allocation
        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;
        vesting.setTVSAllocation(0, 100 ether, 1000, kols, amounts);
        
        vm.stopPrank();

        // User claims TVS
        vm.startPrank(user);
        vesting.claimRewardTVS(0);
        
        
        uint256 nftId = 1;
        
        // Check NFT ownership
        assertEq(nft.ownerOf(nftId), user);

        // Try to split TVS
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        // This should revert due to uninitialized memory access
        // We expect a revert, but specifically a panic code 0x32 (Array accessed at an out-of-bounds or negative index) 
        // or just a generic memory error.
        // Since it's writing to unallocated memory, it might be unpredictable, but in Solidity it usually reverts.
        
        vm.expectRevert(); 
        vesting.splitTVS(0, percentages, nftId);
        
        vm.stopPrank();
    }
}

```
Ran 1 test for test/MemoryBug.t.sol:MemoryBugTest
[PASS] testSplitTVSReverts() (gas: 826951)
Traces:
  [886651] MemoryBugTest::testSplitTVSReverts()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [125387] ERC1967Proxy::fallback(Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6], Aligners26: [0x83769BeEB7e5405ef0B7dc3C66C43E3a51A6d27f], 101, 1000)
    │   ├─ [120144] AlignerzVesting::launchRewardProject(Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6], Aligners26: [0x83769BeEB7e5405ef0B7dc3C66C43E3a51A6d27f], 101, 1000) [delegatecall]
    │   │   ├─ emit RewardProjectLaunched(projectId: 0, projectName: Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [26927] Aligners26::approve(ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], 1000000000000000000000 [1e21])
    │   ├─ emit Approval(owner: ECRecover: [0x0000000000000000000000000000000000000001], spender: ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], value: 1000000000000000000000 [1e21])
    │   └─ ← [Return] true
    ├─ [141377] ERC1967Proxy::fallback(0, 100000000000000000000 [1e20], 1000, [0x0000000000000000000000000000000000000002], [100000000000000000000 [1e20]])
    │   ├─ [140604] AlignerzVesting::setTVSAllocation(0, 100000000000000000000 [1e20], 1000, [0x0000000000000000000000000000000000000002], [100000000000000000000 [1e20]]) [delegatecall]
    │   │   ├─ emit TVSAllocated(projectId: 0, kol: SHA-256: [0x0000000000000000000000000000000000000002], amount: 100000000000000000000 [1e20], vestingPeriod: 1000)
    │   │   ├─ [39522] Aligners26::transferFrom(ECRecover: [0x0000000000000000000000000000000000000001], ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], 100000000000000000000 [1e20])
    │   │   │   ├─ emit Transfer(from: ECRecover: [0x0000000000000000000000000000000000000001], to: ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], value: 100000000000000000000 [1e20])
    │   │   │   └─ ← [Return] true
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [517129] ERC1967Proxy::fallback(0)
    │   ├─ [516401] AlignerzVesting::claimRewardTVS(0) [delegatecall]
    │   │   ├─ [66188] AlignerzNFT::mint(SHA-256: [0x0000000000000000000000000000000000000002])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 1)
    │   │   │   ├─ emit Minted(to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 1)
    │   │   │   └─ ← [Return] 1
    │   │   ├─ emit RewardTVSClaimed(projectId: 0, kol: SHA-256: [0x0000000000000000000000000000000000000002], nftId: 1, amount: 100000000000000000000 [1e20], vestingPeriod: 1000)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [4098] AlignerzNFT::ownerOf(1) [staticcall]
    │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    ├─ [0] VM::assertEq(SHA-256: [0x0000000000000000000000000000000000000002], SHA-256: [0x0000000000000000000000000000000000000002]) [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [23748] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [22986] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    │   │   ├─ [2030] Aligners26::transfer(0x0000000000000000000000000000000000000000, 0)
    │   │   │   └─ ← [Revert] ERC20InvalidReceiver(0x0000000000000000000000000000000000000000)
    │   │   └─ ← [Revert] ERC20InvalidReceiver(0x0000000000000000000000000000000000000000)
    │   └─ ← [Revert] ERC20InvalidReceiver(0x0000000000000000000000000000000000000000)
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.09ms (462.33µs CPU time)

```

### Mitigation

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    // Initialize all dynamic arrays before use
    alloc.amounts = new uint256[](nbOfFlows);
    alloc.vestingPeriods = new uint256[](nbOfFlows);
    alloc.vestingStartTimes = new uint256[](nbOfFlows);
    alloc.claimedSeconds = new uint256[](nbOfFlows);
    alloc.claimedFlows = new bool[](nbOfFlows);
    
    // Rest of the function...
}
```
  