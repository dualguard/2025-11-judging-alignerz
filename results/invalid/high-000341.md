# [000341] Users will pay 0.5% merge fee per TVS when protocol specification says that merging is FREE
  
  ### Summary

Incorrect fee application logic in `mergeTVS` will cause financial loss for users as the protocol applies a 0.5% fee to each source TVS being merged, resulting in cumulative fees of 0.5% × Z (where Z = number of TVSs), when the AlignerZ specification explicitly states that merging should be free.

### Root Cause

In [`AlignerzVesting.sol:mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1021), the function incorrectly applies fees to each source TVS:

```solidity
function mergeTVS(...) external returns(uint256) {
    // ... 

    uint256[] memory amounts = mergedTVS.amounts;
    uint256 nbOfFlows = mergedTVS.amounts.length;
    
    // Fee applied to destination TVS
    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
    mergedTVS.amounts = newAmounts;  // Reduced by 0.5%

    uint256 nbOfNFTs = nftIds.length;
    // ...

    // @audit-isseu: fee also applied to each source TVS in _merge()
    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(...);
    }    
    // ...
}

function _merge(...) internal returns (uint256 feeAmount) {
    // ...
    for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
        uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);  // Another 0.5% fee!
        mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
        // ...
        feeAmount += fee;
    }
    // ...
}

### Internal Pre-conditions

1. User must own multiple TVS NFTs (destination and source TVSs)
2. All TVSs must be from the same project with matching token types
3. User must call `mergeTVS` which in turn calls `_merge`, a helper function, to combine their TVSs
4. Contract must have sufficient token balance to transfer fees to treasury

### External Pre-conditions

_No response_

### Attack Path

1. Alice owns TVS A with 1,000 tokens and TVS B with 2,000 tokens (total: 3,000 tokens)
2. Alice calls `mergeTVS(projectId, nftA, [projectId], [nftB])` to merge TVS B into TVS A
3. The function applies 0.5% fee to TVS A: 1,000 × 0.5% = 5 tokens deducted
4. TVS A now has 995 tokens after destination fee
5. The function then processes TVS B through `_merge()` 
6. Inside `_merge()`, another 0.5% fee is applied to TVS B: 2,000 × 0.5% = 10 tokens deducted
7. TVS B's remaining 1,990 tokens are appended to TVS A
8. Total fees charged: 5 + 10 = 15 tokens (0.5% of total 3,000 tokens)
9. Alice's merged TVS has only 2,985 tokens instead of 3,000 tokens
10. Alice has lost 15 tokens in fees when merging should be FREE per specification
11. The 15 tokens are permanently transferred to treasury

### Impact

Users suffer direct financial loss of 0.5% of their total TVS value for each merge operation. In a scenario where a user merges 10 TVSs (each containing 1,000 tokens, total 10,000 tokens), the cumulative fee would be 50 tokens (0.5% of 10,000), and this should be free according to the specification. This misalignment with the specification creates a significant trust issue for users.

### PoC

For this PoC, I creater a `MockAlignerzVesting` contract to help with `setAllocation()` logic and also a `getAllocation()` helper function
```solidity


        /*//////////////////////////////////////////////////////////////
                        MERGE FEE BUG TEST
    //////////////////////////////////////////////////////////////*/
contract MergeTVSTest is Test {
    MockAlignerzNFT public nft;
    MockERC20 public token;
    MockAlignerzVesting public vesting;
    
    address public alice = address(0x1);
    address public bob = address(0x2);
    address public treasury = address(0x999);
    
    uint256 constant PROJECT_ID = 1;
    uint256 constant YEAR = 365 days;
    uint256 constant BASIS_POINT = 10000;
    
    function setUp() public {
        nft = new MockAlignerzNFT("A26Z NFT", "A26ZN", 18);
        token = new MockERC20("A26Z", "A26Z", 18);
        vesting = new MockAlignerzVesting(address(nft), treasury);
        
        vesting.createBiddingProject(PROJECT_ID, address(token), address(token));

        token.mint(address(vesting), 1_000_000 * 1e18);
        
        vm.label(alice, "Alice");
        vm.label(treasury, "Treasury");
    }

    function test_MultipleFeeApplication() public {        
        // Alice has TVS A (destination)
        uint256 nftA = nft.mint(alice);
        
        uint256[] memory amountsA = new uint256[](1);
        amountsA[0] = 1000 * 1e18;
        
        uint256[] memory periods = new uint256[](1);
        periods[0] = YEAR;
        
        uint256[] memory starts = new uint256[](1);
        starts[0] = block.timestamp;
        
        vesting.setAllocation(PROJECT_ID, nftA, true, amountsA, periods, starts);
        
        // Alice has TVS B (to merge)
        uint256 nftB = nft.mint(alice);
        
        uint256[] memory amountsB = new uint256[](1);
        amountsB[0] = 2000 * 1e18;
        
        vesting.setAllocation(PROJECT_ID, nftB, true, amountsB, periods, starts);
        
        // Alice has TVS C (to merge)
        uint256 nftC = nft.mint(alice);
        
        uint256[] memory amountsC = new uint256[](1);
        amountsC[0] = 3000 * 1e18;
        
        vesting.setAllocation(PROJECT_ID, nftC, true, amountsC, periods, starts);
        
        uint256 treasuryBefore = token.balanceOf(treasury);
        
        console.log("TVS A (destination): 1,000 tokens");
        console.log("TVS B (to merge): 2,000 tokens");
        console.log("Total: 3,000 tokens\n");
        
        // Merge B & C into A
        uint256[] memory projectIds = new uint256[](2);
        projectIds[0] = PROJECT_ID;
        projectIds[1] = PROJECT_ID;
        
        uint256[] memory nftIds = new uint256[](2);
        nftIds[0] = nftB;
        nftIds[1] = nftC;
        
        vm.prank(alice);
        vesting.mergeTVS(PROJECT_ID, nftA, projectIds, nftIds);
        
        uint256 treasuryAfter = token.balanceOf(treasury);
        uint256 feeCharged = treasuryAfter - treasuryBefore;
        
        (uint256[] memory finalAmounts,,,,,,,) = vesting.getAllocation(PROJECT_ID, nftA, true);
        
        uint256 totalAfterMerge = 0;
        for (uint i = 0; i < finalAmounts.length; i++) {
            totalAfterMerge += finalAmounts[i];
        }
        
        console.log("=== RESULTS ===");
        console.log("Fee charged:", feeCharged);
        console.log("Total tokens after merge:", totalAfterMerge);
        console.log("Expected total: 6,000 tokens");
        console.log("Loss to fees:", 6000 * 1e18 - totalAfterMerge);

        assertGt(feeCharged, 0, "BUG: Fee charged when merging should be FREE");
    }

// CReated this `MockAlignerzVesting` contract to get a good flow of how the actual contract works
// Everything is the same, But I only included the `setAllocation()` logic for clarity.
contract MockAlignerzVesting {

    /**
        * @notice Sets the allocation for a given NFT in a project.
        * @param projectId The ID of the project.
     */
    function setAllocation(
        uint256 projectId,
        uint256 nftId,
        bool isBidding,
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes
    ) external {
        Allocation storage alloc;

        if (isBidding) {
            alloc = biddingProjects[projectId].allocations[nftId];
        } else {
            alloc = rewardProjects[projectId].allocations[nftId];
        }

        alloc.amounts = amounts;
        alloc.vestingPeriods = vestingPeriods;
        alloc.vestingStartTimes = vestingStartTimes;
        alloc.claimedSeconds = new uint256[](amounts.length);
        alloc.claimedFlows = new bool[](amounts.length);
        alloc.isClaimed = false;

        if (isBidding) {
            alloc.token = biddingProjects[projectId].token;
        } else {
            alloc.token = rewardProjects[projectId].token;
        }

        NFTBelongsToBiddingProject[nftId] = isBidding;
        allocationOf[nftId] = alloc;
    }
}
```

### Mitigation

Remove all fee calculations from the `mergeTVS` function to align with the AlignerZ specification:

```diff
function mergeTVS(...) external returns(uint256) {
    // ...
    uint256 nbOfNFTs = nftIds.length;
    // ...

    // No fees applied - merging is FREE per spec
-    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
-   mergedTVS.amounts = newAmounts; 
    for (uint256 i; i < nbOfNFTs; i++) {
+        _merge(...);
-       feeAmount += _merge(...);
    }
    
    // ...
}

function _merge(...) internal {  // No longer returns feeAmount
    // ...
    uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
    
    for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
        // Remove fee calculation
-        uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
-        mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
+        mergedTVS.amounts.push(TVSToMerge.amounts[j]);
// ...    
}
```
  