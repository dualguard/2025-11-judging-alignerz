# [000954] Duplicate KOL Entries Cause Incorrect Reward Allocation and Orphaned Tokens
  
  ### Summary

The `setTVSAllocation` and `setStablecoinAllocation` functions allow the admin to add the same KOL address multiple times during reward configuration. Since the mapping `kolTVSRewards[kol]` is address-based, each duplicate entry overwrites the previous allocation amount. However, the contract deposits the full sum of all amounts (including duplicates), creating a mismatch where only the last assigned amount is stored and claimable, while excess tokens remain orphaned in the contract.

### Root Cause

`AlignerzVesting.sol#L458-L477` and `setStablecoinAllocation`

The function:
1. Does not check for duplicate addresses in the `kolTVS` array
2. Pushes all addresses (including duplicates) to `kolTVSAddresses` array
3. Overwrites `kolTVSRewards[kol]` for duplicate addresses (only last value persists)
4. Sums all amounts including duplicates in `totalAmount`
5. Transfers the full `totalTVSAllocation` to the contract

### Internal Pre-conditions

1. Owner must call `setTVSAllocation` or `setStablecoinAllocation` with duplicate KOL addresses in the input arrays
2. The duplicate addresses must have different allocation amounts to demonstrate the issue clearly
3. The `totalTVSAllocation` parameter must equal the sum of all amounts (including duplicates)


### External Pre-conditions

-

### Attack Path

1. Admin launches a reward project via `launchRewardProject`
2. Admin calls `setTVSAllocation` with:
   - `kolTVS = [Alice, Alice]`
   - `TVSamounts = [100, 200]`
   - `totalTVSAllocation = 300`
3. Contract state after execution:
   - `kolTVSRewards[Alice] = 200` (second value overwrites first)
   - `kolTVSAddresses = [Alice, Alice]` (both entries stored)
   - Contract receives 300 tokens
4. Alice claims via `claimRewardTVS`:
   - Receives NFT with allocation of 200 tokens
   - `kolTVSRewards[Alice]` set to 0
5. Result:
   - Alice can only claim 200 tokens
   - 100 tokens remain stuck in contract
   - Owner can withdraw stuck tokens via `withdrawStuckTokens`


### Impact

1. **Incorrect reward distribution**: KOLs receive less than the intended total allocation
2. **Orphaned tokens**: Unclaimable tokens accumulate in the contract (difference between sum of duplicates and last value)
3. **Centralization risk**: Owner can extract leftover tokens via `withdrawStuckTokens`, creating trust issues
4. **Allocation unreliability**: The reward system becomes unreliable and error-prone
5. **Financial loss**: KOLs lose entitled rewards if admin makes configuration errors

### PoC

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import "../src/contracts/nft/AlignerzNFT.sol";
import "../src/contracts/token/Aligners26.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract DuplicateKolBugTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    Aligners26 stablecoin;
    address owner = address(0x1);
    address user = address(0x2);

    function setUp() public {
        vm.startPrank(owner);
        nft = new AlignerzNFT("AlignerzNFT", "ANFT", "https://example.com/");
        AlignerzVesting vestingImpl = new AlignerzVesting();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(vestingImpl),
            abi.encodeWithSelector(AlignerzVesting.initialize.selector, address(nft))
        );
        vesting = AlignerzVesting(payable(address(proxy)));
        nft.addMinter(address(vesting));
        token = new Aligners26("Token", "TKN");
        stablecoin = new Aligners26("Stable", "STB");
        vm.stopPrank();
    }

    function testDuplicateKolAllocationLoss() public {
        vm.startPrank(owner);
        vesting.launchRewardProject(address(token), address(stablecoin), block.timestamp, 1000);
        
        uint256 amount1 = 100 ether;
        uint256 amount2 = 200 ether;
        uint256 totalAmount = amount1 + amount2;
        
        token.approve(address(vesting), totalAmount);
        
        address[] memory kols = new address[](2);
        kols[0] = user;
        kols[1] = user; // Duplicate
        
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = amount1;
        amounts[1] = amount2;
        
        // This should pass, transferring 300 ether to contract
        vesting.setTVSAllocation(0, totalAmount, 1000, kols, amounts);
        vm.stopPrank();

        // User claims
        vm.startPrank(user);
        vesting.claimRewardTVS(0);
        
        // User should have received an NFT. Let's check the amount in the NFT allocation.
        // We expect it to be amount2 (200), and amount1 (100) is lost.
        // NFT ID should be 0 (or 1 depending on counter).
        uint256 nftId = 0;
        try nft.ownerOf(0) returns (address) { nftId = 0; } catch { nftId = 1; }
        
        // (uint256[] memory allocAmounts,,,,,,,) = vesting.allocationOf(nftId);
        // assertEq(allocAmounts[0], amount2, "User got the second amount");
        
        // Try to claim again?
        vm.expectRevert(); // Should revert because allowance is cleared
        vesting.claimRewardTVS(0);
        
        vm.stopPrank();
        
        // Check contract balance. Should be 300.
        // User claimed NFT, but tokens are still in contract until claimed from NFT.
        // claimRewardTVS MINTs the NFT. It doesn't transfer tokens yet.
        // So contract balance should be 300.
        assertEq(token.balanceOf(address(vesting)), totalAmount);
        
        // Now user claims tokens from NFT
        vm.startPrank(user);
        vm.warp(block.timestamp + 2000); // Vesting over
        vesting.claimTokens(0, nftId);
        
        // User balance should be amount2 (200)
        assertEq(token.balanceOf(user), amount2);
        
        // Contract balance should be amount1 (100) -> Stuck!
        assertEq(token.balanceOf(address(vesting)), amount1, "Amount1 is stuck in contract");
        
    }
}
```

```
[PASS] testDuplicateKolAllocationLoss() (gas: 1015702)
Traces:
  [1075402] DuplicateKolBugTest::testDuplicateKolAllocationLoss()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [125387] ERC1967Proxy::fallback(Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6], Aligners26: [0x83769BeEB7e5405ef0B7dc3C66C43E3a51A6d27f], 1, 1000)
    │   ├─ [120144] AlignerzVesting::launchRewardProject(Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6], Aligners26: [0x83769BeEB7e5405ef0B7dc3C66C43E3a51A6d27f], 1, 1000) [delegatecall]
    │   │   ├─ emit RewardProjectLaunched(projectId: 0, projectName: Aligners26: [0x2c1DE3b4Dbb4aDebEbB5dcECAe825bE2a9fc6eb6])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [26927] Aligners26::approve(ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], 300000000000000000000 [3e20])
    │   ├─ emit Approval(owner: ECRecover: [0x0000000000000000000000000000000000000001], spender: ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], value: 300000000000000000000 [3e20])
    │   └─ ← [Return] true
    ├─ [189321] ERC1967Proxy::fallback(0, 300000000000000000000 [3e20], 1000, [0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000002], [100000000000000000000 [1e20], 200000000000000000000 [2e20]])
    │   ├─ [188536] AlignerzVesting::setTVSAllocation(0, 300000000000000000000 [3e20], 1000, [0x0000000000000000000000000000000000000002, 0x0000000000000000000000000000000000000002], [100000000000000000000 [1e20], 200000000000000000000 [2e20]]) [delegatecall]
    │   │   ├─ emit TVSAllocated(projectId: 0, kol: SHA-256: [0x0000000000000000000000000000000000000002], amount: 100000000000000000000 [1e20], vestingPeriod: 1000)
    │   │   ├─ emit TVSAllocated(projectId: 0, kol: SHA-256: [0x0000000000000000000000000000000000000002], amount: 200000000000000000000 [2e20], vestingPeriod: 1000)
    │   │   ├─ [39522] Aligners26::transferFrom(ECRecover: [0x0000000000000000000000000000000000000001], ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], 300000000000000000000 [3e20])
    │   │   │   ├─ emit Transfer(from: ECRecover: [0x0000000000000000000000000000000000000001], to: ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], value: 300000000000000000000 [3e20])
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
    │   │   ├─ emit RewardTVSClaimed(projectId: 0, kol: SHA-256: [0x0000000000000000000000000000000000000002], nftId: 1, amount: 200000000000000000000 [2e20], vestingPeriod: 1000)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [1460] AlignerzNFT::ownerOf(0) [staticcall]
    │   └─ ← [Revert] OwnerQueryForNonexistentToken()
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [3354] ERC1967Proxy::fallback(0)
    │   ├─ [2622] AlignerzVesting::claimRewardTVS(0) [delegatecall]
    │   │   └─ ← [Revert] Caller_Has_No_TVS_Allocation()
    │   └─ ← [Revert] Caller_Has_No_TVS_Allocation()
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [1306] Aligners26::balanceOf(ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc]) [staticcall]
    │   └─ ← [Return] 300000000000000000000 [3e20]
    ├─ [0] VM::assertEq(300000000000000000000 [3e20], 300000000000000000000 [3e20]) [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [0] VM::warp(2001)
    │   └─ ← [Return]
    ├─ [149428] ERC1967Proxy::fallback(0, 1)
    │   ├─ [148697] AlignerzVesting::claimTokens(0, 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    │   │   ├─ [44229] AlignerzNFT::burn(1)
    │   │   │   ├─ emit Approval(owner: SHA-256: [0x0000000000000000000000000000000000000002], approved: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   │   │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   │   │   └─ ← [Return]
    │   │   ├─ [29844] Aligners26::transfer(SHA-256: [0x0000000000000000000000000000000000000002], 200000000000000000000 [2e20])
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc], to: SHA-256: [0x0000000000000000000000000000000000000002], value: 200000000000000000000 [2e20])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit TokensClaimed(projectId: 0, isBiddingProject: false, poolId: 0, isClaimed: true, nftId: 1, claimedSeconds: [1000], claimTimestamp: 2001, user: SHA-256: [0x0000000000000000000000000000000000000002], amount: [200000000000000000000 [2e20]])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [1306] Aligners26::balanceOf(SHA-256: [0x0000000000000000000000000000000000000002]) [staticcall]
    │   └─ ← [Return] 200000000000000000000 [2e20]
    ├─ [0] VM::assertEq(200000000000000000000 [2e20], 200000000000000000000 [2e20]) [staticcall]
    │   └─ ← [Return]
    ├─ [1306] Aligners26::balanceOf(ERC1967Proxy: [0xc051134F56d56160E8c8ed9bB3c439c78AB27cCc]) [staticcall]
    │   └─ ← [Return] 100000000000000000000 [1e20]
    ├─ [0] VM::assertEq(100000000000000000000 [1e20], 100000000000000000000 [1e20], "Amount1 is stuck in contract") [staticcall]
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 26.12ms (2.20ms CPU time)
```

### Mitigation

Implement duplicate address detection in both `setTVSAllocation` and `setStablecoinAllocation`:
  